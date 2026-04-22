# Полная выдержка диалога и результата исследования по `ryanmcdermott/markus`

## Проверенные источники

- Репозиторий: https://github.com/ryanmcdermott/markus
- Основной заголовочный файл: https://github.com/ryanmcdermott/markus/blob/main/markus.h
- CLI: https://github.com/ryanmcdermott/markus/blob/main/main.cc
- BUILD-файл: https://github.com/ryanmcdermott/markus/blob/main/BUILD
- Статья PVS-Studio на Хабре: https://habr.com/ru/companies/pvs-studio/articles/1026084/

---

# 1. Диалог

## Пользователь

Original custom instructions no longer available

## Система

Information about the original user was redacted.

---

## Пользователь

https://github.com/ryanmcdermott/markus

Можешь провести полную аналитику и оценку репозитория с точки зрения оптимизации, качества кода, ошибок, неточностей и псевдо-оптимизаций. Оценивай только основной код, остальные зависимости не важны, также забей на тесты. Выдели точки, где ты видишь проблемы, а не просто давай общие слова

---

## Пользователь

Пройдись по всему коду и выдели все проблемные места опираясь на твои рекомендации. Мне нужна таблица проблем и способов решения

## Ассистент

Соберу это в прикладную таблицу: файл/участок, проблема, чем опасно, как исправить и приоритет. Чтобы не размазывать, буду оценивать только `markus.h` и `main.cc`, без тестов/зависимостей.

## Ассистент

Я уже вижу несколько не просто «код-стайл», а реальных архитектурных рисков: глобальный PMR-ресурс, частично валидируемый UTF-8/Unicode, дублирующее хранение AST и CLI, который молча игнорирует неизвестные аргументы. Сейчас упаковываю это в таблицу без общих фраз.

## Ассистент

Ниже — таблица именно по проблемным местам основного кода. Я не трогаю тесты и внешние зависимости. Оценка статическая: без запуска бенчмарков и санитайзеров.

Контекст: проект сам описан как single-header C++20 Markdown parser, а README прямо предупреждает “DO NOT USE IN PRODUCTION” и указывает, что производительность сейчас в 2–3 раза ниже `cmark`. Основной файл — `markus.h`: 6483 строки, 191 KB; CLI — `main.cc`, 51 строка.

| # | Место | Проблема | Почему плохо | Как исправить | Приоритет |
|---:|---|---|---|---|---|
| 1 | `markus.h`: глобальный `g_buffer[128MB]`, `g_arena`, `set_default_resource` | Глобальный allocator меняет default PMR resource для всей программы. | Любой чужой `std::pmr::*` в процессе может начать использовать allocator Markus. Это скрытый глобальный сайд-эффект от `#include "markus.h"`. | Убрать `std::pmr::set_default_resource`; завести `ParseContext` / `ParserOptions` с локальным `memory_resource*`; передавать allocator явно. | Critical |
| 2 | `markus.h`: тот же global arena | Один общий `monotonic_buffer_resource` не потокобезопасен для параллельного парсинга. | В веб-сервисе/многопоточном приложении возможны гонки и порча состояния allocator. | Делать arena per-parse/per-thread; либо `thread_local`, но лучше контекст парсинга без глобального состояния. | Critical |
| 3 | `markus.h`: фиксированный `128 MiB` buffer | Память резервируется статически даже для маленького Markdown. | Псевдо-оптимизация: ускоряет часть аллокаций, но делает библиотеку тяжёлой по памяти и неудобной для embedding. | Начать с маленького stack/heap buffer, масштабировать по размеру input; дать пользователю настраивать arena size. | High |
| 4 | `markus.h`: `monotonic_buffer_resource` | Нет явного сброса памяти после одного парсинга. | При многократных вызовах в одном процессе память будет расти до лимита arena/heap spill, даже если документы уже не нужны. | Локальный `monotonic_buffer_resource` внутри `Parse()`; после завершения render/AST lifetime — destroy/release context. | High |
| 5 | `markus.h`: header-only с глобальной инициализацией | Простое подключение заголовка выполняет глобальную настройку allocator. | Header-only библиотека не должна менять глобальное состояние приложения при include. | Оставить header-only, но всю инициализацию делать только при вызове API, через объект контекста. | Critical |
| 6 | `Document`: одновременно `children`, `inline_nodes`, `block_nodes` | У AST две модели хранения: дерево через `children` и node pools через ID. | Легко получить рассинхрон: часть кода может читать `children`, часть — pools. Плюс лишняя память. | Выбрать одну модель: либо arena/pool + IDs, либо обычный ownership tree. Для compact storage лучше оставить pools и сделать `children` только ID-list. | High |
| 7 | `AddBlock`: `BlockNodeId id = static_cast<BlockNodeId>(block_nodes.size())` | Нет проверки переполнения `uint32_t`. | На экстремально больших входах возможен wrap ID и обращение не к тому узлу. | Перед cast проверять `size() <= numeric_limits<BlockNodeId>::max()`; либо заменить ID на `size_t`/`uint64_t`. | Medium |
| 8 | `Text`, `HtmlInline`: `std::string_view`; другие узлы — owning `pmr::string` | Смешанная модель владения строками. | Нужно строго гарантировать lifetime исходного input и `string_storage`. Любой рефактор легко сломает dangling views. | Документировать lifetime или хранить offsets в исходном input; для декодированных строк — единый intern/storage API. | High |
| 9 | `Paragraph`/`Heading`: `raw_content` остаётся в AST | `raw_content` помечен как temporary storage, но живёт в структуре узла. | После inline parsing это лишнее дублирование текста. На больших документах память растёт без пользы. | После построения inline children очищать `raw_content`; лучше держать raw text только во временном parser state. | Medium |
| 10 | `markus.h`: 6483 строки в одном файле | Монолит: AST, parser, renderer, Unicode, HTML escaping, CLI API — всё в одном заголовке. | Сложнее ревьюить, профилировать, изолировать hot paths и править ошибки без побочных эффектов. | Разделить на внутренние модули: `ast`, `block_parser`, `inline_parser`, `html_renderer`, `unicode`, `arena`; при необходимости генерировать amalgamated header. | High |
| 11 | `FindFirstSpecialChar`, `FindFirstUnsafeChar` | Ручные 8-byte loops под “auto-vectorization”. | Это выглядит как SIMD, но фактически остаётся branchy scalar-кодом. Может мешать компилятору и ухудшает читаемость. | Сравнить с простым loop, `memchr`/`std::find_if`, SIMD intrinsics только после benchmark. | Medium |
| 12 | `ScanForLinesAndNulls` | Возможный double-count для `\r\n` на границе 8-byte блока. Код считает `\r`, а `\n` “будет обработан в следующей итерации”, но не пропускает его. | Количество строк может быть неверным на входах с CRLF, если пара попадает на границу блока. Это уже не стиль, а логическая ошибка. | Упростить single-pass scanner: при `\r` сразу проверять следующий байт независимо от chunk boundary и увеличивать индекс. | High |
| 13 | `IsSpanBlank`, `FindNextLineEnding` | Та же 8-byte развёртка с вложенными циклами. | Сложность выше, выгода не доказана. Для коротких строк Markdown overhead может быть больше выигрыша. | Сделать простой сканер; оставить оптимизированный вариант только если benchmark показывает выигрыш. | Medium |
| 14 | `DecodeUtf8At` | UTF‑8 декодируется без проверки continuation bytes, overlong sequences, surrogate range. | Невалидный UTF‑8 может превращаться в некорректные code points, влияя на emphasis/link-label logic. | Полная UTF‑8 валидация: проверка `10xxxxxx`, overlong, `U+D800..DFFF`, `> U+10FFFF`; invalid → U+FFFD с корректным шагом. | High |
| 15 | `IsUnicodePunctuation` | В коде прямо указано, что это не полная Unicode category table. | CommonMark-совместимость для Unicode punctuation будет частичной; возможны ошибки emphasis parsing на не-ASCII текстах. | Использовать сгенерированную таблицу Unicode categories или компактные range tables из UnicodeData. | Medium |
| 16 | `IsUnicodeWhitespaceAt` | Для non-ASCII whitespace явно обрабатывается только NBSP. | Название обещает Unicode whitespace, но покрытие узкое. Это источник тихих расхождений. | Либо переименовать в `IsNbspOrAsciiWhitespaceAt`, либо реализовать полный набор Unicode Zs/line separators. | Medium |
| 17 | `FoldCodePoint` / case folding | Case folding реализован вручную частичными диапазонами. | Link label normalization может быть неверной для многих языков и спец-символов. | Использовать generated Unicode case folding table; отдельно обрабатывать multi-codepoint folds. | Medium |
| 18 | `IsAsciiPunctuation`: 256-byte table | Таблица вручную зашита в код. | Работает быстро, но плохо проверяется глазами; ошибка в одном байте даст трудноуловимый баг. | Генерировать table compile-time через `constexpr` predicate или добавить static tests на диапазоны ASCII punctuation. | Low |
| 19 | `TryParseAutolink` / regex-подход | `std::regex` для URI/email — тяжёлый hot-path для inline parsing. В заголовке подключён `<regex>`. | `std::regex` в C++ часто дорогой по CPU/инициализации; для автоссылок достаточно deterministic scanner. | Заменить на ручной scanner: scheme length/range, `:`, запрет whitespace/`<>`; email — один `@`, доменные метки, длины. | High |
| 20 | README performance claims vs code | README обещает arena allocator, lookup tables, SIMD-friendly algorithms. | Часть “оптимизаций” не доказана: arena глобальная, 8-byte loops спорные, Unicode tables частичные. | Ввести benchmark suite: small/medium/large docs, Unicode-heavy docs, autolink-heavy docs; сравнивать с `cmark`. | High |
| 21 | `main.cc`: `ReadStdin()` читает весь stdin в `std::stringstream` | CLI всегда полностью буферизует input. | Для больших файлов память дублируется: stdin buffer → markdown string → AST/raw_content/render output. | Для CLI хотя бы `std::string input((istreambuf_iterator<char>...))`; для библиотеки рассмотреть streaming/tokenization или reserve по file size. | Medium |
| 22 | `main.cc`: неизвестные аргументы игнорируются | В цикле обрабатываются только `--help` и `--ast`; остальные аргументы не вызывают ошибку. | Опечатка вроде `--ats` silently даст HTML вместо AST. | Добавить `else { std::cerr << "Unknown option"; return 2; }`. | Medium |
| 23 | `main.cc`: нет обработки исключений | `Parse`, `DebugAst`, `MarkdownToHtml` вызываются напрямую. | Любое исключение завершит процесс без нормальной диагностики. | Обернуть main body в `try/catch(const std::exception&)`, писать в stderr, возвращать non-zero. | Medium |
| 24 | `main.cc`: usage говорит только `[--ast]`, хотя есть `--help` | Usage неполный. | Мелочь, но CLI выглядит недоделанным. | `Usage: markus [--ast] [--help]`; добавить `--version` по желанию. | Low |
| 25 | `BUILD`: нет C++20 флагов | Bazel target просто объявляет `cc_library` и `cc_binary`; README требует C++20. | Сборка зависит от внешней настройки toolchain; у пользователя может быть непредсказуемый compile error. | Добавить `copts = ["-std=c++20"]` или toolchain/features config; минимум указать в `.bazelrc`. | Medium |
| 26 | Public API: `Parse`, `RenderHtml`, `MarkdownToHtml`, `DebugAst` | API простой, но без options/config. README перечисляет только базовые функции. | Нельзя настроить allocator, отключить raw HTML, выбрать dialect, включить diagnostics. | Добавить `ParserOptions` / `RenderOptions`: allocator, raw_html_policy, diagnostics, max_input_size. | High |
| 27 | Raw HTML support | README явно заявляет raw inline HTML и HTML blocks. | Это нормально для CommonMark, но опасно для untrusted input: библиотека не является HTML sanitizer. | В README/API явно написать “not sanitizer”; добавить option `escape_raw_html` / `disable_raw_html`. | High |
| 28 | Ошибки/diagnostics | API возвращает готовый `Document`/HTML без видимого канала diagnostics. | При странном input невозможно понять, где парсер “съел” конструкцию или fallback’нулся. | Возвращать `ParseResult { Document doc; vector<Diagnostic> diagnostics; }`; для совместимости оставить старый API. | Medium |

### Что исправлять первым

1. **Глобальный PMR allocator**: убрать `set_default_resource`, сделать per-parse context.
2. **CRLF bug в `ScanForLinesAndNulls`**: конкретная логическая ошибка.
3. **UTF‑8 validation + Unicode tables**: заявленная поддержка Unicode шире фактической.
4. **Autolink regex**: заменить на scanner и замерить.
5. **AST ownership model**: выбрать единую модель хранения.

Главный вывод: проблема Markus не только в отдельных микрооптимизациях, а в смешении библиотечного API с глобальными runtime-решениями. Для CLI это терпимо; для использования как встраиваемой Markdown-библиотеки — рискованно.

---

## Пользователь

https://habr.com/ru/companies/pvs-studio/articles/1026084/

Проверь, не использовал ли ты эту статью. Сравни тезисно проблемы и решения, описанные в статье и в твоей выкладке

## Ассистент

Нет, в моей предыдущей выкладке я **не использовал эту статью как источник**: в ответе были ссылки на GitHub/README/исходники, а не на Хабр. Но есть пересечение по одной центральной теме: **код выглядит оптимизированным, но часть “оптимизаций” является псевдо-оптимизацией**.

### Сравнение тезисов

| Тема | Что говорит статья | Что было у меня | Совпадение |
|---|---|---|---|
| `IsSpanBlank` / “SIMD-friendly” 8-byte loop | Статья подробно разбирает `IsSpanBlank`: вложенный цикл по 8 байт выглядит как SIMD-friendly, но фактически обрабатывает байты последовательно; в бенчмарках простой цикл оказался быстрее, а SSE-вариант — сильно быстрее. | Я отметил `IsSpanBlank`, `FindFirstSpecialChar`, `FindFirstUnsafeChar`, `ScanForLinesAndNulls` как ручные 8-byte loops под “auto-vectorization”, назвал это псевдо-SIMD и предложил сравнить с простым loop/`memchr`/настоящими intrinsics. | **Сильное совпадение** |
| Временные строки при case-insensitive compare | Статья показывает создание `std::pmr::string upper` для проверки `<!DOCTYPE` и предлагает `starts_with_insensitive` без временной строки. | Я это **не выделил**. | **Пропущено мной** |
| HTML block tags: `type1_tags`, `open_tag`, `lower` | Статья разбирает `static const std::pmr::vector<std::pmr::string> type1_tags`, создание `"<" + tag`, затем ещё одной lowercase-строки; предлагает `constexpr std::array<std::string_view>` с готовыми префиксами. | Я это **не выделил**. У меня был общий пункт про raw HTML и API-безопасность, но не про неэффективный парсинг HTML-тегов. | **Пропущено мной** |
| ODR-риск `static` внутри inline-функции | Статья упоминает предупреждение PVS-Studio V1096: `type1_tags` со static storage duration внутри inline-функции может привести к ODR violation. | Я это **не выделил**. | **Пропущено мной** |
| `std::variant` как тяжёлое хранение inline nodes | Статья подробно объясняет, что `InlineNode = std::variant<...>` раздувается до размера самой большой альтернативы; из-за `Image` с тремя `std::pmr::string` один `InlineNode` может быть около 128 байт, даже для `SoftBreak`. Это ухудшает расход памяти и cache locality. | Я отметил дублирующее хранение AST и смешанную модель `children`/node pools, но **не выделил конкретно размер `std::variant` и проблему 128 байт на inline node**. | **Частично пересекается, но статья точнее** |
| Лишний `NodeType` / `kType` | Статья говорит, что `NodeType`/`kType` фактически лишние: `kType` не используется, а `NodeTypeToString` мало полезен. | Я это **не выделил**. | **Пропущено мной** |
| `push_back` вместо настоящего in-place construction | Статья разбирает PVS V823: простая замена на `emplace_back(Text(...))` почти ничего не даст; для `std::variant` нужно `emplace_back(std::in_place_type<Text>, ...)`. | Я это **не выделил**. | **Пропущено мной** |
| Копия вместо move | Статья указывает на копирование `content` в `heading.raw_content`, где можно `std::move(content)` или сразу вернуть aggregate. | Я упоминал `raw_content` как лишнее временное хранилище в AST, но не эту конкретную копию. | **Частично** |
| Двойной lookup в unordered_map | Статья показывает `find` + `insert`, где лучше `try_emplace`, чтобы не делать два поиска. | Я это **не выделил**. | **Пропущено мной** |
| Бессмысленные проверки / “склероз” | Статья приводит `!input.empty()` как всегда true после раннего выхода и `prev_line_had_content = true` внутри ветки, где оно уже true. | Я это **не выделил**. | **Пропущено мной** |
| Глобальный PMR allocator / `set_default_resource` | В найденной статье этого тезиса нет. | У меня это был главный критический пункт: глобальный allocator и общий arena как риск для библиотеки. | **Мой отдельный тезис** |
| UTF‑8 / Unicode correctness | В статье нет подробного разбора UTF‑8/Unicode. | У меня были пункты про неполную UTF‑8 validation, Unicode punctuation/whitespace/case folding. | **Мой отдельный тезис** |
| CLI / BUILD / API options | Статья почти полностью про `markus.h` и PVS/эффективность; CLI, Bazel и публичные options не являются её фокусом. | У меня были отдельные пункты про `main.cc`, неизвестные аргументы, исключения, C++20 флаги, `ParserOptions`. | **Мои отдельные тезисы** |

### Итог

Моя выкладка **не была пересказом статьи**: я не затронул несколько её самых характерных находок — временные строки в HTML parsing, `std::variant` размером около 128 байт, лишние `NodeType/kType`, `try_emplace`, `std::in_place_type`, PVS-предупреждения.

Но статья **усиливает и уточняет** один мой важный пункт: “SIMD-friendly” и ручные микрооптимизации действительно выглядят как псевдо-оптимизации. Там это подтверждено бенчмарками, а у меня было сформулировано как статическое подозрение.

Если обновить мою таблицу после статьи, я бы добавил как минимум эти новые строки:

| Приоритет | Новая проблема | Решение |
|---|---|---|
| High | `IsSpanBlank` 8-byte loop хуже простого цикла | Заменить на простой loop или настоящий SIMD после benchmark |
| High | `std::variant<InlineNode>` раздувает каждый inline node до размера самой большой альтернативы | Пересмотреть layout: pool/IDs, compact node representation, tagged union с external storage для тяжёлых вариантов |
| Medium | `std::pmr::string upper/lower` для case-insensitive prefix checks | Ввести `starts_with_insensitive(std::string_view, std::string_view)` |
| Medium | `type1_tags` как `pmr::vector<pmr::string>` внутри inline-функции | Заменить на `static constexpr std::array<std::string_view, N>` |
| Medium | `find` + `insert` в map | Заменить на `try_emplace` |
| Low/Medium | `push_back(Text(...))` для `std::variant` | Использовать `emplace_back(std::in_place_type<Text>, args...)` |
| Low | Лишние `NodeType/kType` | Удалить или оставить только если реально нужен unified debug/type API |
| Low | Бессмысленные проверки и присваивания | Почистить по PVS-предупреждениям |

Так что корректная картина такая: **моя таблица шире по архитектуре и API, статья глубже по конкретным C++-неэффективностям внутри `markus.h`.**

---

## Пользователь

Дай файл с полной выдержкой нашего диалога в md формате

## Ассистент

Сделаю Markdown-файл с видимой частью диалога: твои запросы и мои ответы, без служебных/скрытых сообщений инструментов.

## Ассистент

Готово: [скачать Markdown-файл с выдержкой диалога](sandbox:/mnt/data/dialogue_markus_analysis.md).

---

## Пользователь

А результат ресёрча ты почему не добавил? Его тоже надо

## Ассистент

Да, это мой промах: я сохранил видимую переписку, но не добавил отдельный блок с результатом ресёрча. Сейчас сделаю обновлённый `.md`: диалог + отдельная секция “Результат исследования” с тезисами, таблицей проблем и сравнением со статьёй.

---

# 2. Результат исследования

## 2.1. Что именно проверялось

Проверялся только основной код проекта Markus:

- `markus.h` — основной single-header parser/rendering/AST код.
- `main.cc` — простой CLI.
- `BUILD` — минимальная Bazel-конфигурация.

Не оценивались:

- тесты;
- внешние зависимости;
- CommonMark fixtures;
- стиль как самоцель, если он не ведёт к ошибкам, лишней памяти, рискам API или псевдо-оптимизациям.

## 2.2. Ключевые факты из README/репозитория

По README проект заявлен как:

- single-header C++20 Markdown parser;
- zero dependencies;
- CommonMark-compliant;
- использующий arena allocator, string views, lookup tables, SIMD-friendly algorithms и compact node storage;
- поддерживающий Unicode: UTF-8 encode/decode, case folding, punctuation и whitespace handling.

При этом README прямо предупреждает:

- `DO NOT USE IN PRODUCTION`;
- код “vibe coded”;
- не проходил полноценное review на memory safety;
- производительность примерно в 2–3 раза ниже `cmark`.

Это важно: часть заявленных оптимизаций уже конфликтует с собственным предупреждением проекта о текущей незрелости.

## 2.3. Главная картина по качеству

Основная проблема Markus — не один конкретный баг, а сочетание четырёх факторов:

1. **Глобальные runtime-решения внутри header-only библиотеки.**  
   Глобальный PMR allocator и preallocated arena опасны для встраивания в чужие приложения.

2. **Псевдо-оптимизации.**  
   В коде есть ручные 8-byte loops, “SIMD-friendly” сканеры, lookup tables и PMR-структуры, но без доказанной пользы и местами с худшей читаемостью/надёжностью.

3. **Частичная Unicode-корректность при широких заявлениях.**  
   README заявляет full Unicode support, но ручные таблицы/функции выглядят неполными.

4. **Неустойчивый AST/data layout.**  
   Одновременно используются pool-based storage, ID-шники, `std::variant`, raw_content и string views. Это создаёт риски лишней памяти, dangling references и рассинхронизации.

## 2.4. Основные проблемы и решения

| # | Категория | Проблема | Риск | Решение | Приоритет |
|---:|---|---|---|---|---|
| 1 | Архитектура | Глобальный `std::pmr::set_default_resource` | Влияет на весь процесс, не только на Markus | Убрать глобальный default resource; передавать allocator через context/options | Critical |
| 2 | Архитектура | Общий global arena | Потенциальные гонки в многопоточном использовании | Arena per-parse/per-context | Critical |
| 3 | Память | 128 MiB static buffer | Лишняя память даже для маленьких документов | Настраиваемый buffer, lazy allocation, per-document arena | High |
| 4 | Память | `monotonic_buffer_resource` без понятного release на parse boundary | Рост памяти при многократном парсинге | Локальный parse context и явный lifetime | High |
| 5 | Header-only API | Include меняет глобальное состояние | Неприемлемо для библиотеки | Инициализация только при вызове API | Critical |
| 6 | AST | Смешение `children`, `inline_nodes`, `block_nodes` | Возможна рассинхронизация дерева и pools | Единая модель: IDs/pools или ownership tree | High |
| 7 | AST | `uint32_t` IDs без overflow check | Wrap на больших документах | Проверки или `size_t`/`uint64_t` | Medium |
| 8 | Lifetime | `string_view` + owning strings в разных узлах | Dangling views при изменении lifetime input/storage | Offsets в input или единый string storage | High |
| 9 | Память | `raw_content` остаётся в AST | Дублирование текста | Держать raw text только во временном parser state | Medium |
| 10 | Поддерживаемость | 6483 строки в одном header | Сложно ревьюить и профилировать | Разбить на модули и при необходимости делать amalgamated header | High |
| 11 | Псевдо-оптимизация | 8-byte loops в scanners | Сложнее, не факт что быстрее | Простые loops/`memchr`/SIMD только после benchmark | Medium/High |
| 12 | Возможный баг | `ScanForLinesAndNulls` и CRLF на границе блока | Неверный подсчёт строк | Single-pass scanner с корректным пропуском `\n` после `\r` | High |
| 13 | Unicode | `DecodeUtf8At` без полной валидации | Некорректные code points | Полная UTF-8 validation + U+FFFD fallback | High |
| 14 | Unicode | Частичные punctuation/whitespace/case folding tables | Ошибки CommonMark на non-ASCII | Generated Unicode tables | Medium/High |
| 15 | Performance | `std::regex` в autolinks | Дорогой hot path | Ручной deterministic scanner | High |
| 16 | Security/API | Raw HTML без явного sanitizer disclaimer/options | XSS при untrusted input | Опции disable/escape raw HTML + README warning | High |
| 17 | CLI | Неизвестные аргументы игнорируются | Ошибки пользователя проходят молча | Error on unknown option | Medium |
| 18 | CLI | Нет обработки исключений | Плохая диагностика | `try/catch` в `main` | Medium |
| 19 | Build | Нет явного C++20 в BUILD | Сборка зависит от внешней конфигурации | Добавить C++20 feature/copts | Medium |
| 20 | API | Нет `ParserOptions`/`RenderOptions` | Нельзя настраивать allocator, raw HTML, diagnostics | Ввести options и `ParseResult` | High |

## 2.5. Дополнения после статьи PVS-Studio

Статья PVS-Studio добавляет более точные C++-level проблемы, которые в первой таблице были не полностью раскрыты:

| Проблема из статьи | Почему важно | Решение |
|---|---|---|
| `IsSpanBlank` “SIMD-friendly” loop хуже простого цикла | Ручная микрооптимизация не просто бесполезна, а может проигрывать | Заменить на простой loop или настоящий SIMD после benchmark |
| Временный `std::pmr::string upper` для проверки `<!DOCTYPE` | Аллокации/копии на hot path | `starts_with_insensitive(std::string_view, std::string_view)` |
| `type1_tags` как `std::pmr::vector<std::pmr::string>` | Дорого и рискованно для header-only/inline | `constexpr std::array<std::string_view>` |
| ODR-риск static object внутри inline-функции | Потенциальная проблема линковки/ODR | `inline constexpr` storage или namespace-scope constexpr table |
| `std::variant<InlineNode>` раздувает каждый inline node | Плохая cache locality и лишняя память | Compact tagged union, pools, external storage для тяжёлых вариантов |
| Лишние `NodeType` / `kType` | Мёртвый metadata-шум | Удалить или реально использовать в unified debug API |
| `push_back(Text(...))` вместо in-place variant construction | Лишние временные объекты | `emplace_back(std::in_place_type<Text>, args...)` |
| Копирование `content` в `heading.raw_content` | Лишняя копия | `std::move(content)` или aggregate return |
| `find` + `insert` в map | Двойной lookup | `try_emplace` |
| Бессмысленные проверки/присваивания | Шум и риск маскировки логических ошибок | Удалить по результатам static analysis |

## 2.6. Сравнение моей оценки и статьи

| Тема | Моя оценка | Статья PVS-Studio | Итог |
|---|---|---|---|
| Псевдо-SIMD / 8-byte loops | Отмечено как подозрительная псевдо-оптимизация | Подтверждено бенчмарками на `IsSpanBlank` | Сильное совпадение |
| Глобальный PMR allocator | Центральный критический риск | Не основной фокус статьи | Мой отдельный тезис |
| Unicode correctness | Отдельный блок проблем | Не основной фокус статьи | Мой отдельный тезис |
| AST/data layout | Отмечено через смешение pools/tree/raw_content | Уточнено через размер `std::variant` | Статья делает тезис точнее |
| HTML parsing allocations | Не выделено изначально | Подробно разобрано | Пропуск в моей первой версии |
| `try_emplace`, `in_place_type`, move | Не выделено изначально | Подробно разобрано | Пропуск в моей первой версии |
| CLI/API/BUILD | Отдельно выделено | Почти не рассматривается | Мой отдельный тезис |
| Security/raw HTML | Отмечено как API/security риск | Не основной фокус статьи | Мой отдельный тезис |

## 2.7. Приоритизированный план исправлений

### Этап 1. Убрать архитектурные риски

1. Удалить глобальный `set_default_resource`.
2. Ввести `ParseContext`.
3. Сделать allocator/arena локальным для одного parse/render цикла.
4. Добавить `ParserOptions`.

### Этап 2. Починить корректность

1. Переписать line scanner для CRLF.
2. Усилить UTF-8 validation.
3. Пересмотреть Unicode punctuation/whitespace/case folding tables.
4. Добавить diagnostics.

### Этап 3. Почистить псевдо-оптимизации

1. Заменить 8-byte loops простыми baseline-версиями.
2. Сделать benchmark suite.
3. Возвращать ручную оптимизацию только там, где есть измеримый выигрыш.
4. `std::regex` заменить на scanner.

### Этап 4. Оптимизировать memory layout

1. Пересмотреть `std::variant` для inline nodes.
2. Убрать лишний `raw_content`.
3. Определиться с единой моделью AST.
4. Сделать compact representation для частых маленьких узлов.

### Этап 5. Улучшить CLI/build

1. Unknown CLI option должен возвращать ошибку.
2. Добавить `try/catch` в `main`.
3. Явно зафиксировать C++20 в сборке.
4. Добавить usage/version.

## 2.8. Сводный вывод

Markus выглядит как проект, где “оптимизированный C++” во многом имитируется внешними признаками:

- PMR;
- arena allocator;
- 8-byte scanning;
- lookup tables;
- `std::string_view`;
- compact IDs;
- `[[likely]]`/`[[unlikely]]`-style подходы;
- single-header minimalism.

Но реальная оценка показывает:

1. **PMR использован опасно**, потому что задевает глобальное состояние процесса.
2. **Arena allocator не делает код автоматически быстрым**, особенно при 128 MiB static buffer и неочевидном lifetime.
3. **8-byte scanning не равен SIMD** и может быть хуже простого цикла.
4. **`string_view` не бесплатен**, если lifetime не формализован.
5. **Compact node IDs не гарантируют compact AST**, если рядом лежит тяжёлый `std::variant`.
6. **Full Unicode support заявлен сильнее, чем выглядит реализация.**
7. **Raw HTML требует явной политики безопасности.**

Наиболее корректная формулировка: Markus — интересный экспериментальный parser, но сейчас он не выглядит как production-ready библиотека. Главные проблемы — глобальное состояние, спорные микрооптимизации, неполная Unicode-корректность и неустойчивый memory/data layout.
