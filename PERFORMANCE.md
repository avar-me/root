# Производительность словаря (ru.avar.me и клоны)

Рекомендации для разработчиков движка (шаблоны, `build_data.py`, фронтенд).  
Первичные замеры: 13–14 сентября 2026. Повторный аудит прода: **14 сентября 2026** (`build_id` ru ≈ `b3d1c9628538`).

Сайт — статический. Оболочка лёгкая; узкое место — **объём индексов словаря** до полной интерактивности поиска. Часть сетевых рекомендаций уже на проде (см. [Статус на проде](#статус-на-проде)).

---

## Контекст стека

```
браузер → Cloudflare (DNS + HTTPS + кэш) → GitHub Pages (Fastly) → статика из docs/
```

- Сборка: `build.sh` → `src/build_data.py` + `src/apply_profile.py`
- Фронт: `src/templates/html/{index.html,app.js,styles.css}`, фразы — `phrases.html` / `phrases.js`
- Cache-bust: `?v=__ASSET_VERSION__` (= `build_id` из `manifest.json`)

**Данные словаря в этот репозиторий не коммитить** — правки смыслов в `avar-me/sources`. Здесь только формат чанков, индексов и клиентская логика загрузки.

Внешний вид без явного запроса не менять (см. `CLAUDE.md`).

---

## Статус на проде

Аудит живых `ru` / `en` / `tr` / `uk` (+ HTML-заголовки `de` / `fr` / `be`), 14 сентября 2026.

| # | Рекомендация | Статус | Комментарий |
|---|--------------|--------|-------------|
| — | CF Cache Rules на всех языковых хостах | **Сделано** | `ru en de fr tr uk be`. Data и HTML: после прогрева PoP — `HIT` (раньше data было `DYNAMIC`) |
| 1 | `Promise.all` в `init` / `switchDictType` | **Сделано** | Параллельная загрузка пяти индексов; то же при смене словаря |
| 2 | Поиск доступен до конца загрузки индексов | **Открыто** | Поле в DOM видно, но `initEventListeners` + focus — только после `Promise.all` |
| 3 | Убрать Universal Analytics | **Сделано** | UA/`analytics.js` нет; GA4 тоже нет |
| 4 | Lazy `browse.json` | **Открыто** | По-прежнему в стартовом `Promise.all` (~575 КБ gzip av-ru) |
| 5 | Lazy `form_to_headword` | **Открыто** | То же (~222 КБ gzip) |
| 6 | Preload согласовать с загрузкой | **Открыто** | Preload только `index.words.txt` + `browse.json` |
| 7 | Чанки без `indent=2` | **Сделано** | Compact JSON одной строкой |
| 8 | Ужать schema `manifest.json` | **Открыто** | Поля `words_count` / `size` / `hash` на месте; из‑за большего числа чанков gzip манифеста вырос (~59 КБ) |
| 9 | Сплит крупных префиксов | **Сделано** (по симптомам) | av-ru: **~1444 → ~2986** чанков; медиана ~19 КБ → ~9.7 КБ; гигантского `гӏа.json` ~1.87 МБ больше нет |
| 10 | Phrases: не грузить всё до ввода | **Частично** | Очередь+ретраи против Safari; объём на старте тот же |
| 11–12 | IndexedDB / idle-prefetch 2-го словаря | **Открыто** | Нет |
| 13 | Self-host шрифтов | **Открыто** | Google Fonts как раньше |

**Эффект уже на проде (Европа, прогретый CF):**

- Пять индексов параллельно: wall **~1.2–1.4 с** (раньше цепочка ~2.8 с).
- Повторный визит: data с края CF (`HIT`), не `DYNAMIC` через GitHub Pages.
- Меньше сырого объёма чанков статей (compact + finer split).

**Следующий заметный выигрыш:** lazy `browse` + `form_to_headword` и проводка обработчиков поиска до их загрузки (P1 + п. 2).

---

## Cloudflare (инфра)

Cache Rules на зоне (хосты словарей):

**Data** — Eligible for cache, длинный Edge TTL:

```text
(http.host in {"ru.avar.me" "en.avar.me" "de.avar.me" "fr.avar.me" "tr.avar.me" "uk.avar.me" "be.avar.me"} and starts_with(http.request.uri.path, "/data/"))
```

**HTML** — Eligible for cache, короткий Edge TTL (~1 ч; у HTML нет `?v=`):

```text
(http.host in {"ru.avar.me" "en.avar.me" "de.avar.me" "fr.avar.me" "tr.avar.me" "uk.avar.me" "be.avar.me"} and (
  http.request.uri.path eq "/" or
  ends_with(http.request.uri.path, ".html")
))
```

Проверка:

```bash
curl -sI -H "Accept-Encoding: gzip" \
  "https://ru.avar.me/data/av-ru/browse.json?v=<build_id>" \
  | grep -iE 'cf-cache-status|age|cache-control'

curl -sI "https://ru.avar.me/" | grep -iE 'cf-cache-status|age|cache-control'
```

После деплоя новый `?v=` даёт холодный `MISS` на data — ожидаемо; второй запрос в том же PoP — `HIT`.  
Кэш на краю **не убирает** скачивание ~1 МБ gzip индексов на первый визит в PoP.

---

## Замеры (ориентиры)

Оболочка (gzip/br):

| Ресурс | Transfer |
|--------|----------|
| `index.html` | ~3.3 КБ (br) |
| `styles.css` + `theme.css` | ~5.4 КБ |
| `app.js` | ~11 КБ gzip (~40 КБ сырой после правок) |

Старт главной **av-ru** до конца `init()` — по-прежнему **пять файлов** (gzip):

| Файл | gzip | сырой | Роль |
|------|------|-------|------|
| `browse.json` | ~575 КБ | ~3.3 МБ | глоссы главной / таблицы префикса |
| `form_to_headword.json` | ~222 КБ | ~2.0 МБ | форма → lemma |
| `index.words.txt` | ~213 КБ | ~1.4 МБ | префиксный индекс |
| `index.headwords.txt` | ~85 КБ | ~0.4 МБ | статьи, рандом на главной |
| `manifest.json` | ~59 КБ | больше прежнего | ~2986 чанков (было ~29 КБ / ~1444) |
| **сумма** | **~1.15 МБ** | **~7+ МБ** | `initEventListeners` после полного `Promise.all` |

- Чанки статей (после сплита/compact): av-ru **~2986**, медиана **~9.7 КБ** (раньше ~1444 / ~19 КБ; выброс `гӏа` ~1.87 МБ устранён).
- `phrases.html`: оба направления целиком до ввода (~3.8 МБ gzip) — без изменений по модели загрузки.
- Европа, прогретый CF: пять файлов **параллельно ~1.2–1.4 с** (было последовательно ~2.8 с).

Шрифты: Google Fonts (Onest + Literata), render-blocking CSS.

---

## Критический путь сейчас

1. HTML + render-blocking fonts CSS + styles.
2. `app.js` в конце `body`.
3. `init()` грузит **параллельно** (`Promise.all`):  
   `words` + `headwords` + `form_to_headword` + `browse` + `manifest`.
4. Пока `Promise.all` не завершится — нет `initEventListeners` / focus; спиннер у блока результатов.
5. Preload только у `index.words.txt` и `browse.json`.

`switchDictType()` — тот же параллельный набор для второго словаря (у ru-av `browse.json` ~977 КБ gzip).

---

## Рекомендации по приоритету

### P0 — быстрые правки в клиенте (без смены дизайна)

#### 1. Параллельная загрузка индексов — сделано

`Promise.all` в `init()` и `switchDictType()` на проде. Ориентир: цепочка ~2.8 с → параллель ~1.2–1.4 с.

#### 2. Не блокировать UI до конца загрузки индексов — открыто

Поле поиска в разметке видно сразу, но `initEventListeners()` и `focus` вызываются только после `Promise.all`. Печать до готовности индексов не обрабатывается.

Нужно: повесить слушатели раньше; запросы буферизовать до готовности `wordsIndex` / `manifest`; индикатор — у подсказок / результатов / «случайных слов», не как единственный смысл экрана.

#### 3. Убрать Universal Analytics — сделано

UA/`analytics.js` с прода сняты. При возврате аналитики — только async/defer (GA4 и т.п.), не sync в `<head>`.

---

### P1 — меньше байт на критическом пути

#### 4. Снять `browse.json` со старта

Файл нужен для глосс в таблице префикса и ~14 случайных слов на главной (`HOME_SAMPLES`), **не** для открытия поля ввода.

Варианты:

- грузить `browse.json` лениво при первом префиксном поиске / после `requestIdleCallback`;
- или на сборке писать маленький `samples.json` (N статей с глоссами) для главной, а полный `browse` — по требованию.

**Эффект:** минус ~575 КБ gzip (av-ru) / ~977 КБ (ru-av) до интерактива.

**Где:** `app.js` + при необходимости `src/build_data.py` (`write_browse`).

#### 5. Отложить `form_to_headword.json`

Точный lookup и манифест/чанки могут работать раньше. Карта форм нужна для префикса по словоформам и резолва в lemma.

Грузить при первом вводе длиной ≥ 1–2 символа или после готовности базового индекса.

**Эффект:** минус ~222 КБ gzip до первого осмысленного поиска.

#### 6. Preload согласовать с реальной загрузкой — открыто

Сейчас preload только `index.words.txt` + `browse.json`. После п. 4–5: не preload’ить lazy-файлы; для оставшихся в стартовом `Promise.all` — preload всех или ни одного лишнего.

---

### P2 — сборка данных

#### 7. Чанки без `indent=2` — сделано

Чанки на проде — compact JSON. Медиана сырого размера заметно ниже.

#### 8. Ужать `manifest.json` — открыто

Клиенту для `findChunkForWord` достаточно `prefix` + `file`. Сейчас в манифесте ещё `words_count`, `size`, `hash`, … После finer split число чанков выросло (~2986) → gzip манифеста ~59 КБ (было ~29 КБ при ~1444).

Compact + урезанный schema; `build_id` считать на сборке. Смена schema = согласование с `app.js` / TMA и ключом кэша (`build_id`).

#### 9. Сплит крупных префиксов — сделано (по симптомам)

Число чанков av-ru ~2986, медиана ~9.7 КБ; прежний выброс `гӏа.json` ~1.87 МБ не наблюдается. Имеет смысл оставить регрессионную проверку в сборке: падать/логировать, если чанк > `MAX_CHUNK_SIZE` / `MAX_WORDS_PER_CHUNK`.

---

### P3 — phrases и повторные визиты

#### 10. `phrases.html`: не качать всё до ввода — частично

`loadPhraseSet` по-прежнему тянет **все** чанки обоих словарей (~54 файла, ~3.8 МБ gzip) до снятия спиннера — модель «всё заранее» открыта.

**Сделано (надёжность):** Safari/iOS при `Promise.all` на все чанки сразу сыпал `Failed to load resource: The network connection was lost`. В `phrases.js` — очередь с `CHUNK_CONCURRENCY: 4`, ретраи обрывов, словари грузятся последовательно (не два пула параллельно).

Дальше: грузить по вводу / с прогрессом / не блокировать поле на весь объём.

#### 11. Клиентский кэш по `build_id`

В документации уже предполагался кэш, завязанный на `build_id`. В `app.js` сейчас только in-memory `chunkCache` (LRU 50).

IndexedDB или Cache Storage: индексы + чанки, ключ = `build_id` / `ASSET_VERSION`. Второй визит без сети на индексы.

#### 12. Prefetch второго словаря на idle

После успешной загрузки av-ru — `requestIdleCallback` / `prefetch` для ru-av (или наоборот), чтобы `switchDictType` не повторял полный wait.

---

### Осторожно / продуктово

#### 13. Self-host шрифтов

Сейчас `fonts.googleapis.com` + `fonts.gstatic.com`. Self-host 2–3 woff2 (кириллица + латиница, нужные веса) + `preload` убирает render-blocking на Google.

Согласовать с `index.avar.me` / дизайн-системой. Без запроса на UI — не менять произвольно.

#### 14. Не ломать нормализацию

`normalize_word()` в `build_data.py` и `normalizeWord()` / `normalizeQuery()` во фронте должны оставаться согласованными (палочка → U+04CF, ё→е). Иначе ключи чанков разъедутся с поиском.

---

## Что не является приоритетом

- Минификация `app.js` / `styles.css` — уже малы (~11 / ~5 КБ gzip).
- HTTP/2 / HTTP/3 — уже есть у Cloudflare.
- SSR / npm / бандлер — сознательно нет; не вводить без сильной причины.
- Кешировать HTML на неделю на CF без purge — опасно из‑за залипания `__DICT_ASSET_V__`.

---

## Предлагаемый порядок работ

| Шаг | Задача | Где | Статус |
|-----|--------|-----|--------|
| — | CF Cache Rules на всех языковых хостах | Cloudflare | **Сделано** |
| 1 | `Promise.all` в `init` / `switchDictType` | `app.js`, TMA | **Сделано** |
| 2 | Поиск доступен сразу, спиннер локальный | `app.js` | Открыто |
| 3 | Убрать UA / не блокировать head | `index.html`, `phrases.html` | **Сделано** |
| 4 | Lazy `browse.json` (+ опционально `samples.json`) | `app.js`, `build_data.py` | Открыто ← следующий фокус |
| 5 | Lazy `form_to_headword` | `app.js` | Открыто |
| 6a | Compact chunks | `build_data.py` | **Сделано** |
| 6b | Slim manifest | `build_data.py`, клиент | Открыто |
| 7 | Починить oversized chunk (гӏа) | `build_data.py` | **Сделано** (симптомы) |
| 8 | Phrases: очередь чанков + ретраи (Safari); lazy по вводу | `phrases.js` | Очередь **сделана**, lazy открыто |
| 9 | IndexedDB по `build_id` | `app.js` | Открыто |

---

## Критерии приёмки (коротко)

- [x] Сетевой waterfall: индексы стартуют **параллельно**, не цепочкой.
- [x] Повторный визит: `cf-cache-status: HIT` на `/data/*` (инфра, все языковые хосты).
- [x] Нет sync Universal Analytics в `<head>`.
- [ ] После загрузки HTML обработчики поиска / focus **до** завершения скачивания всех индексов (достаточно `words`+`manifest` для базового сценария).
- [ ] В DevTools (cold, disable cache): transfer до интерактива заметно меньше ~1.1 МБ (без `browse` и по возможности без `form_to_headword` на старте).
- Регрессии: поиск по формам/палочке/ё, переключение словаря, `#word=` / `#dict=`, phrases, TMA.

---

## Ссылки

- Инструкции по репо: [`CLAUDE.md`](CLAUDE.md)
- Источник данных: [sources.avar.me](https://sources.avar.me) / [avar-me/sources](https://github.com/avar-me/sources)
- Живой сайт: [ru.avar.me](https://ru.avar.me)
