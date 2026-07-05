# MinecraftJarMigrator

Инструмент для автоматической миграции jar-файлов модов и плагинов Minecraft между версиями.

```
input.jar → BytecodeTransform (ASM) → Remap (Tiny Remapper) → Decompile → Transform (JavaParser) → Compile → output.jar
```

Поддерживает 6 загрузчиков: **Fabric**, **Forge**, **NeoForge**, **Quilt**, **Paper**, **Spigot**.
Мигрирует с версий **1.14.4** до **26.2** через **903 набора JSON-правил** для 42 релизов.

---

## Быстрый старт

### GUI (без аргументов)

```bash
java -jar migrator.jar
# Или: двойной клик на MinecraftJarMigrator.exe
```

Откроется окно с drag-and-drop, автоматическим анализом мода и пошаговым пайплайном.

### CLI

```bash
java -jar migrator.jar \
  --input my-mod-1.20.3.jar \
  --output my-mod-1.21.1.jar \
  --from 1.20.3 \
  --to 1.21.1 \
  --type fabric
```

---

## Сборка

Требуется **JDK 26+** и интернет (Maven Central + maven.fabricmc.net).

```bash
./gradlew shadowJar
# → build/libs/migrator.jar
```

### Windows .exe (встроенный Java)

```bat
build-exe.bat
```

Скрипт автоматически скачает Gradle 9.6.0 и JDK 26, если не найдены.
Результат: `build\dist\MinecraftJarMigrator\MinecraftJarMigrator.exe` — работает без установленной Java.

---

## Пайплайн миграции

| Этап | Описание |
|---|---|
| **1. Bytecode Transform** | ASM-трансформации: RefmapPatcher (ремап `*-refmap.json`), MixinConfigPatcher (`compatibilityLevel`), AccessWidenerPatcher (`.accesswidener`) |
| **2. Remap** | Tiny Remapper — переименование классов/методов/полей через цепочку `named_old → obf → named_new` (MojMap); поверх — Parchment-имена параметров и javadoc |
| **3. Decompile** | Декомпиляция байткода в Java-исходники (Vineflower / CFR / Procyon / Авто A/B-подбор) |
| **4. Transform** | AST-трансформации через JavaParser + JSON-правила: переименование методов, классов, перенос пакетов |
| **5. Compile** | Рекомпиляция через `javax.tools`. Поблочный фоллбэк: при неудаче пакетной компиляции каждый файл компилируется отдельно |

После миграции рядом с выходным jar создаётся отчёт (`*.report.txt`), а также HTML и Markdown версии.

---

## GUI

Запуск без аргументов или с одним `.jar` («Открыть с помощью»).

### Возможности интерфейса

- **Drag & Drop** — перетащите .jar для автоматического анализа
- **Авто-анализ** — определяет загрузчик (Fabric/Forge/NeoForge/Quilt/Paper/Spigot), версию MC, метаданные мода
- **Информация о моде** — название, ID, версия, MC, загрузчик, авторы, лицензия, описание, библиотеки
- **Выбор декомпилятора** — Vineflower (рекомендуется), CFR, Procyon, **Авто** (A/B-подбор: пробует декомпиляторы на выборке и выбирает с меньшим числом ошибок)
- **Кэш декомпиляции** — SHA-256 кэш в `~/.mcmigrator/decompile-cache/`
- **Прогресс-бары** — поэтапный прогресс, включая счётчик классов внутри декомпиляции
- **Итоговый экран** — крупный вердикт **GO / RISKY / NO-GO** (confidence 0–100) + вердикт запуска `loads: yes/no` и кнопки действий
- **Матрица версий** — раскрашенная шкала целевых версий (зелёный/жёлтый/красный) с ценой прыжка ещё до запуска
- **Темы** — светлая и тёмная (FlatLaf), переключение кнопкой
- **Масштаб** — кнопки A−/A+ для изменения размера шрифта
- **Профили настроек** — сохранение/загрузка конфигураций; поля формы автосохраняются между запусками
- **История миграций** — журнал всех запусков с повтором и сравнением confidence «было/стало»
- **Командная палитра** — Ctrl+K, быстрый доступ ко всем действиям
- **Встроенный редактор кода** — просмотр и правка декомпилированных исходников с подсветкой синтаксиса (RSyntaxTextArea), quick-fix по Alt+Enter, перекомпиляция в jar

### Вкладки

| Вкладка | Описание |
|---|---|
| **Журнал** | Живой лог миграции с кнопкой копирования |
| **Smoke-тест** | Вердикт `loads: yes/no` и лог запуска сервера (при `--verify`) |
| **Отчёт** | Фильтруемый отчёт: трансформации, ручные правки, предупреждения |
| **Изменения (diff)** | Unified diff для каждого изменённого файла |
| **Совместимость** | Матрица совместимости, рекомендации, заметки |
| **Граф зависимостей** | Текстовый граф зависимостей мода |
| **Проверка jar** | Preflight-валидация: дескрипторы, классы, исходники, вложенные jar |
| **Smoke-test** | Генерация скрипта `test-server-{version}.bat` для проверки загрузки мода на сервере |
| **Защита JAR** | Анализ и снятие обфускации/защиты |

### Кнопки

| Кнопка | Действие |
|---|---|
| **Мигрировать** | Запуск полного пайплайна миграции |
| **Проверить Mixin'ы** | Верификация mixin-целей против jar целевой версии |
| **Проверить jar** | Preflight-проверка входного jar |
| **Что изменится?** | Быстрый предпросмотр без декомпиляции: сколько классов затронется и сколько правил сработает |
| **Анализ защиты** | Сканирование JAR на 78 типов обфускации и защиты |
| **Снять защиту** | Автоматическое снятие того, что можно исправить безопасно (см. [«Защита JAR»](#защита-jar)); остальное — только обнаружение с предупреждением |
| **Smoke-test** | Генерация тестового скрипта для Fabric-сервера |
| **Пакетно** | Пакетная миграция всех jar из папки (сводный HTML: jar × confidence × loads) |
| **Обновить модпак** | Скан папки/`.mrpack`: для каждого мода — скачать готовый релиз под целевую версию или мигрировать |
| **История** | Просмотр и повтор предыдущих миграций |
| **Редактировать код** | Встроенный редактор декомпилированных исходников |
| **Копировать журнал** | Копирование лога в буфер обмена |

---

## CLI — все параметры

| Параметр | Описание |
|---|---|
| `-i, --input` | Входной `.jar` (обязательно) |
| `-o, --output` | Выходной `.jar` (обязательно) |
| `--from` | Исходная версия MC, напр. `1.20.3` (обязательно) |
| `--to` | Целевая версия MC, напр. `1.21.1` (обязательно) |
| `--type` | `fabric` / `forge` / `neoforge` / `paper` / `spigot` / `quilt` (по умолчанию: `paper`) |
| `--decompiler` | `VINEFLOWER` / `CFR` / `PROCYON` / `AUTO` (A/B-подбор; по умолчанию: `VINEFLOWER`) |
| `--verify` | После миграции поднять headless-сервер с мод-jar и получить вердикт `loads: yes/no` (Fabric/Quilt/Paper/Forge/NeoForge) |
| `--no-decompile-cache` | Отключить кэширование декомпиляции |
| `--parallel` | Параллельная декомпиляция (только Vineflower) |
| `--interactive` | Интерактивный выбор breaking-правил перед миграцией |
| `--classpath` | Доп. classpath для рекомпиляции (через запятую) |
| `--auto-classpath` | Автоскачивание classpath (по умолчанию: вкл; отключить: `--no-auto-classpath`) |
| `--keep-sources` | Сохранить декомпилированные исходники в `{output}-sources/` |
| `--dry-run` | Показать изменения без записи файлов |
| `--cascade` | Каскадная миграция через промежуточные версии |
| `--batch` | Пакетный режим: `--input` и `--output` — папки |
| `--disable-rule` | Отключить правило по описанию (можно повторять) |
| `--mappings-dir` | Папка кэша маппингов (по умолчанию: `~/.mcmigrator/mappings`) |
| `-v, --verbose` | Подробный вывод (DEBUG) |

---

## Поддерживаемые версии

### Как источник (from)

Минимальная версия: **1.14.4**. Версии до 1.14.4 (1.6.4–1.14.3) отображаются в интерфейсе, но миграция с них заблокирована — нет официальных MojMap, иная экосистема модов.

### Как цель (to)

43 версии: от **1.14.4** до **26.2**

### Версии Java

| MC версия | Мин. Java | Примечание |
|---|---|---|
| 1.14.4 – 1.20.4 | Java 17 | Стандарт |
| 1.20.5 – 1.21.x | Java 21 | Data components |
| 26.1+ (вкл. 26.2) | Java 25 | Year.drop нумерация |

---

## JSON-правила

Правила для пар версий: `src/main/resources/rules/{from}-to-{to}.json`.
**903 набора** для 42 релизов. Генерация: `python3 tools/generate_rules.py`.

```json
{
  "version": "1.20.3-to-1.21.1",
  "methodRenames": [
    { "className": "...", "oldMethod": "...", "newMethod": "...", "comment": "..." }
  ],
  "classRenames": [
    { "oldName": "...", "newName": "..." }
  ],
  "packageMoves": [
    { "oldPackage": "...", "newPackage": "..." }
  ],
  "apiBreakingChanges": [
    { "description": "...", "matchPattern": "regex", "suggestedFix": "..." }
  ]
}
```

---

## Анализ и проверки

| Компонент | Описание |
|---|---|
| **MixinAnalyzer** | Проверяет mixin-цели против jar целевой версии |
| **MixinPortSuggester** | Предлагает кандидатов-замены для сломанных mixin-целей |
| **DependencyChecker** | Валидирует зависимости из `fabric.mod.json`, `mods.toml`, `plugin.yml` |
| **ComplexityAnalyzer** | Оценка сложности миграции: TRIVIAL → EASY → MEDIUM → HARD → MANUAL |
| **MigrationPreview** | «Что изменится?» без декомпиляции: скан constant pool × правила пары |
| **VersionCoverage** | Мгновенная (офлайн) оценка цены прыжка для матрицы версий |
| **MigrationConfidence** | Агрегат сигналов в один балл 0–100 + go/no-go |
| **PostMigrationChecker** | ASM-верификация всех ссылок на классы/методы/поля |
| **PreflightChecker** | Предварительная проверка jar: дескрипторы, классы, исходники, вложенные jar |
| **CompatibilityAdvisor** | Матрица совместимости и рекомендации |
| **LibraryCatalog** | Анализ библиотек мода |

---

## Защита JAR

### Анализ (78 типов обнаружения)

Анализатор обнаруживает: шифрование строк, обфускация управления потоком, переименование классов, reflection, шифрование ресурсов, anti-tamper, native код, custom classloader, ZLIB-упаковка, InvokeDynamic обфускация, обфускация чисел, watermarks, удаление debug-информации, illegal-name обфускация, control flow flattening, dead code, CRC/hash проверки, anti-debug, anti-decompiler, шифрование классов, AES-шифрование, multi-stage loader, и многие другие.

### Снятие защиты

`ProtectionRemover` осознанно делит находки на два класса — реально правит байткод
только там, где это можно сделать безопасно и обобщённо, не гадая об алгоритме
конкретного обфускатора и не исполняя чужой байткод:

**Реально меняет байткод** (каждый класс после этого проходит verify-pass —
структурный dataflow-анализ стека без загрузки классов мода; при провале класс
откатывается к оригинальным байтам с предупреждением, а не тихо ломает jar):
- Переименование классов с недопустимыми/unicode-символами или короткими
  обфусцированными именами — через `SimpleRemapper`/`ClassRemapper` сразу по
  всему jar-у, чтобы ссылки на класс из других классов остались рабочими;
  пропускается, если класс упомянут в текстовом ресурсе (манифест/`mods.toml`/
  `fabric.mod.json`), иначе поломало бы загрузку мода
- Восстановление StackMapTable (`COMPUTE_FRAMES` без загрузки классов мода —
  офлайн-резолвер общего предка по локальной иерархии jar-а)
- Заглушивание native-методов
- Снятие watermark-констант (поля/строки по паттерну хэша, UUID, base64-блоба)
- Расшифровка строк — только детерминированный Base64/hex, без перебора шифров
- Упрощение ветвлений по известной константе (`IFEQ`/`IFNE`) со сбалансированным стеком
- Constant-folding: сворачивает смежные `ICONST/BIPUSH/SIPUSH/LDC` + арифметику/
  битовые операции в одну константу
- Очистка synthetic/bridge флагов, дескрэмбл номеров строк, нейтрализация `setAccessible()`

**Только обнаруживает и предупреждает** (по дизайну: нельзя по одной сигнатуре
отличить защиту от легитимного использования той же самой API, либо небезопасно
менять без исполнения чужого байткода):
- CRC/hash/integrity-проверки (`MessageDigest`/`CRC32`/`ProtectionDomain`) —
  совпадают с обычной проверкой хэша скачанного файла
- Anti-debug (`ManagementFactory`, `attach`, `instrument`) — совпадают с легитимной диагностикой
- Self-modification (`Files`/`FileOutputStream`/`ZipOutputStream`) — обычные API
  сохранения конфигов/ресурсов мода
- `fillInStackTrace()`/`StackTraceElement` — по контракту возвращает `this`,
  реальный код часто использует результат
- InvokeDynamic с кастомным bootstrap-методом — известные JDK-фабрики
  (Lambda/StringConcat/SwitchBootstraps/ObjectMethods/ConstantBootstraps) не считаются подозрительными

---

## Ресурсы

После миграции автоматически патчатся:
- `fabric.mod.json` — обновление `minecraft` зависимости
- `plugin.yml` — обновление версии API
- `mods.toml` — обновление `minecraft` диапазона
- `pack.mcmeta` — обновление `pack_format`

---

## Auto-classpath

Для модов Fabric/Forge/NeoForge/Quilt автоматически:
1. Скачивает ванильный server.jar целевой версии
2. Деобфусцирует его через MojMap
3. Извлекает библиотеки
4. Для Paper/Fabric дополнительно скачивает API
5. Всё кэшируется в `~/.mcmigrator/classpath/{version}/`

---

## GitHub Action

```yaml
- uses: your-org/minecraft-jar-migrator/.github@main
  with:
    input: my-mod-1.20.3.jar
    output: my-mod-1.21.1.jar
    from: '1.20.3'
    to: '1.21.1'
    type: fabric
    decompiler: VINEFLOWER
    cascade: 'false'
    keep-sources: 'false'
```

---

## Дополнительные возможности

- **Верификация запуском (`--verify`)** — после миграции поднимает headless-сервер целевой версии с мод-jar и даёт бинарный вердикт `loads: yes/no` (Fabric/Quilt/Paper готовым сервером, Forge/NeoForge через installer `--installServer`)
- **Migration confidence score** — один агрегат 0–100 + GO/RISKY/NO-GO + топ-3 действия, сведённый из match-rate, сложности, битых ссылок, mixin/зависимостей и вердикта smoke-теста
- **Parchment-имена параметров** — поверх MojMap подмешиваются имена параметров и javadoc: декомпилированный код читается без `var3/var4` (отключить: `-Dmcmigrator.parchment=false`)
- **Модпак-режим** — папка модов или `.mrpack`: по хешу каждого мода определяет, есть ли официальный релиз под целевую версию (скачать) или нужна миграция
- **A/B-подбор декомпилятора** — режим `AUTO`: пробует Vineflower и CFR на выборке крупнейших классов, выбирает с меньшим числом ошибок компиляции
- **Обучение правил** — из диффа символов успешных миграций авто-предлагает переименования для пар версий с бедным набором правил (модерация рейтингом community)
- **Оффлайн-префетч** — скачать MojMap + Parchment + intermediary пары версий заранее, чтобы миграция прошла без интернета
- **Докат каскада** — после падения каскад продолжается с незавершённого шага (готовые промежуточные jar переиспользуются)
- **Каскадная миграция** — большие переходы идут через промежуточные версии
- **Пакетный режим** — миграция всех jar из папки со сводной таблицей (HTML: jar × confidence × loads)
- **Интерактивный выбор правил** — диалог включения/отключения breaking-проверок
- **Git интеграция** — автокоммит на ветке `migrate-{from}-to-{to}`
- **Онлайн-правила** — загрузка свежих правил с GitHub без пересборки
- **Community правила** — система рейтингов (полезно/вредно)
- **Data pack правила** — трансформации рецептов, тегов, таблиц лута
- **Asset правила** — трансформации `sounds.json`, моделей блоков, атласов, шрифтов
- **Architectury поддержка** — детекция `@ExpectPlatform`, `@PlatformOnly`
- **Modrinth + CurseForge API** — автоскачивание Fabric API и зависимостей по ID мода (CurseForge — как fallback, когда на Modrinth сборки нет)
- **Телеметрия** — opt-in анонимная статистика миграций
- **Экспорт отчётов** — TXT, HTML, Markdown
- **Smoke-test** — генерация скрипта для проверки мода на Fabric-сервере
- **Визуальный diff jar** — сравнение двух jar (добавленные/удалённые/изменённые классы)
- **Экспорт патчей** — генерация `.patch` файлов из diff миграции
- **I18n** — русский и английский интерфейс
- **LLM-починка ошибок компиляции** (opt-in) — остаточные ошибки `javac` (API-breaking без JSON-правил) чинятся моделью Claude; включается `-Dmcmigrator.llmFix=true` + переменной `ANTHROPIC_API_KEY`

---

## Архитектура (v4.5)

Код организован в чистую слоистую архитектуру с однонаправленными зависимостями:

```
gui (представление) → application (фасады) → pipeline (стадии) → domain / infra
```

- **gui** — только сборка окна и виджеты; вся оркестрация (запуск пайплайна, доступ
  к сервисам, сборка `MigrationConfig`, история) вынесена в `MigratorController`.
- **application** — фасады `MigrationService` / `AnalysisService` / `ProtectionService` /
  `RulesService`: единственная точка входа в движок, через них ходят и GUI, и CLI.
- **pipeline** — оркестрация стадий (remap → bytecode → decompile → transform → compile).
- **domain / infra** — `analysis`, `mappings`, `rules`, `transform`, `decompiler`,
  `protection`, `report`, `classpath`, `io`, `infra`, `integration`, `version`,
  `common`, `exception`.

Направление зависимостей закреплено тестом-стражем **`ArchitectureTest`**: верхний слой
может зависеть от нижнего, но не наоборот, а `gui` — лист (его импортируют только сам
`gui` и `Main`). Сборка падает при нарушении правила. При добавлении нового top-level
пакета внесите его в карту уровней теста.

---

## Структура проекта

```
src/main/java/com/mcmigrator/
│
├── gui/               # Слой ПРЕДСТАВЛЕНИЯ — только сборка окна
│   ├── MigratorGui        # Swing-оболочка: виджеты, потоки, отрисовка
│   ├── MigratorController # оркестрация: пайплайн, сервисы, MigrationConfig, история
│   ├── UiKit, SystemTheme, PipelineStepBar, SwingLogAppender
│   ├── panel/             # вкладки: InsightsPanel, ProtectionPanel, ProtectionTableModel, PanelHost
│   ├── format/            # чистые форматтеры: InsightFormatter, ProtectionFormatter
│   ├── dialog/            # модальные диалоги: HistoryDialog, InfoDialogs, FixupDialog,
│   │                      # VersionMatrixDialog, ModpackDialog
│   └── editor/            # редактор/браузер кода: CodeBrowserWindow, SourceEditorPanel, JarRecompiler
│
├── application/       # Слой ПРИЛОЖЕНИЯ — фасады, единая точка входа в движок (GUI и CLI)
│   └── MigrationService, AnalysisService, ProtectionService, RulesService
│
├── pipeline/          # Оркестрация стадий миграции
│   └── MigrationPipeline, RemapStage, BytecodeTransformStage, DecompileStage, TransformStage,
│       CompileStage, CascadeMigration, BatchMigration, MigrationConfig, PipelineContext,
│       MigrationOutcome, …
│
├── modpack/           # Модпак-режим: ModpackScanner (папка/.mrpack → скачать/мигрировать)
│
├── analysis/          # Анализ мода (domain)
│   ├── mod/               # ModAnalyzer, LibraryCatalog, PreflightChecker, CompatibilityAdvisor
│   ├── mixin/             # MixinAnalyzer, MixinPortSuggester
│   └── checks/            # ComplexityAnalyzer, DependencyChecker, PostMigrationChecker,
│                          # MigrationPreview, VersionCoverage, FixupResolver
│
├── transform/         # TransformRule, MethodRenameRule, ClassRenameRule, PackageChangeRule,
│   │                  # ApiBreakingChangeRule, ArchitecturyRule, TransformContext
│   ├── bytecode/         # AccessWidenerPatcher, MixinConfigPatcher, RefmapPatcher
│   └── resources/        # AssetTransformer, DataPackTransformer, ResourcePatcher, FabricMetadataPatcher
│
├── mappings/          # MappingsProvider, MappingsDownloader, MappingChain, ParchmentProvider, AutoRuleGenerator
├── classpath/         # ClasspathProvider, ModrinthResolver, CurseForgeResolver, AccessWidenerApplier
├── decompiler/        # DecompilerProvider, DecompilerType, DecompilerSelector (A/B), ParallelDecompiler, DecompilerPluginLoader
├── protection/        # ProtectionAnalyzer, ProtectionRemover, ProtectionFeatureScanner
├── report/            # MigrationReport, HtmlReportExporter, MigrationConfidence
├── rules/             # VersionRuleSet, RulePluginLoader, OnlineRulesHub, CommunityRulesHub,
│                      # InteractiveRuleSelector, RuleUpdater, RuleLearner
├── io/                # JarUtil, JarDiffUtil, DiffUtil, TempDirManager
├── infra/             # Telemetry, MigrationHistory, SettingsProfiles, UiPrefs
├── integration/       # GitIntegrator, SmokeTestRunner, ServerSmokeVerifier, TestServerScriptGenerator
├── llm/               # LlmClient, LlmCompileFixer (опц. LLM-починка ошибок компиляции)
├── i18n/              # I18n
├── version/           # MinecraftVersionSupport
├── common/            # Throwables
├── exception/         # MigrationException, RemapException, DecompileException,
│                      # CompileException, MappingsException, TransformException
├── Constants.java
├── Main.java          # композиционный корень (CLI + поднятие GUI)
└── ModType.java
```

---

## Ограничения

- **Цепочка маппингов** — `named_old → obf → named_new`: покрываются только совпадающие obf-имена
- **Auto-classpath** — скачивает ванильный server.jar + библиотеки; для сложных модов добавляйте через `--classpath`
- **MojMap** — лицензия Mojang: учитывайте условия при распространении результатов
- **JavaParser** — pretty-printer переформатирует изменённые файлы; оригинальное форматирование не сохраняется
- **Разрешение типов** — в MethodRenameRule носит эвристический характер; неподтверждённые типы помечаются в отчёте
- **Скачивание маппингов** — требует доступа к `launchermeta.mojang.com` и `piston-data.mojang.com`

---

## Зависимости

| Библиотека | Версия | Назначение |
|---|---|---|
| Picocli | 4.7.7 | CLI парсинг аргументов |
| Vineflower | 1.12.0 | Декомпилятор (основной) |
| CFR | 0.152 | Декомпилятор (альтернативный) |
| Procyon | 0.6.0 | Декомпилятор (альтернативный) |
| Tiny Remapper | 0.13.1 | Ремаппинг байткода |
| Mapping IO | 0.6.1 | Чтение/запись маппингов |
| JavaParser (symbol-solver) | 3.28.2 | AST-трансформации + разрешение типов |
| Gson | 2.14.0 | JSON парсинг |
| SLF4J + Logback | 2.0.18 / 1.5.34 | Логирование |
| FlatLaf | 3.7.1 | GUI темы |
| RSyntaxTextArea | 3.6.2 | Подсветка синтаксиса |
| AutoComplete | 3.3.1 | Автодополнение в редакторе кода |
| LanguageSupport | 3.3.0 | Поддержка Java в редакторе (Ctrl+Space, squigglies) |
| ASM (core, commons, tree, analysis) | 9.9.1 | Байткод-трансформации и verify-pass в `ProtectionRemover` |
| JUnit Jupiter | 6.1.0 | Тесты |

---

## Лицензия

**MIT License** — см. [LICENSE](LICENSE).

## Ответственное использование

Инструмент предназначен для миграции модов/плагинов, **на которые у вас есть право на изменение**: собственные проекты или моды с лицензией, допускающей модификацию. Перед миграцией чужого мода:

- Проверьте лицензию мода
- Предпочитаю спросить автора или использовать официальную сборку для целевой версии
- Сохраняйте атрибуты: не удаляйте имена авторов, кредиты, лицензии из результата
