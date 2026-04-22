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
