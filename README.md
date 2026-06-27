# MinecraftJarMigrator

Инструмент для автоматической миграции (портинга) jar-файлов модов и плагинов Minecraft между версиями.

```
input.jar → BytecodeTransform (ASM) → Remap (Tiny Remapper) → Decompile → Transform (JavaParser) → Compile → output.jar
```

Поддерживает 6 загрузчиков: **Fabric**, **Forge**, **NeoForge**, **Quilt**, **Paper**, **Spigot**.
Мигрирует с версий **1.14.4** до **26.1.2** через **861 набор JSON-правил** для 42 релизов.

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

Требуется **JDK 26+** и интернет (Maven Central + maven.fabricmc.net) для первой сборки.

```bash
./gradlew shadowJar
# → build/libs/migrator.jar
```

### Windows .exe (встроенный Java)

```bat
build-exe.bat
```

Скрипт автоматически скачает Gradle 9.6.0 и JDK 26, если не найдены, и **переиспользует** уже распакованный JDK при повторных запусках (без удаления занятых файлов).
Результат: `build\dist\MinecraftJarMigrator\MinecraftJarMigrator.exe` — работает без установленной Java.

---

## Пайплайн миграции

| Этап | Описание |
|---|---|
| **1. Bytecode Transform** | ASM-трансформации: RefmapPatcher (ремап `*-refmap.json`), MixinConfigPatcher (`compatibilityLevel`), AccessWidenerPatcher (`.accesswidener`) |
| **2. Remap** | Tiny Remapper — переименование классов/методов/полей. По умолчанию через **стабильный intermediary-namespace** (`named_old → intermediary → named_new`); с авто-откатом на цепочку через obf (`named_old → obf → named_new`, MojMap) |
| **3. Decompile** | Декомпиляция байткода в Java-исходники (Vineflower / CFR / Procyon), с кэшем и опциональной параллельной декомпиляцией |
| **4. Transform** | AST-трансформации через JavaParser + JSON-правила (**параллельно** по ядрам, с сохранением форматирования через LexicalPreservingPrinter) |
| **5. Compile** | Рекомпиляция через `javax.tools` под Java **целевой** версии MC. Умный фоллбэк: при ошибках собирается максимально возможное подмножество файлов |

После миграции рядом с выходным jar создаётся отчёт (`*.report.txt`, а также HTML и Markdown). В начале отчёта — секция **«Сводка»**: match-rate маппингов, тайминги этапов, целевая Java и пост-проверка ссылок.

---

## Маппинги и intermediary-бридж

Ремап строит цепочку из двух деревьев маппингов (для исходной и целевой версии).

- **По умолчанию** версии соединяются по **intermediary-именам Fabric** — они стабильны между релизами, поэтому покрытие классов высокое. Замеры (obf-join → intermediary): `1.20.1→1.21.1` 69 %→94 %, `1.20.1→1.20.4` 69 %→97 %, `1.21.1→1.21.4` 70 %→98 %, `1.16.5→1.20.1` 70 %→83 %. На очень больших скачках (`1.14.4→1.21.1`) выигрыш мал — там предел задаёт не имена, а отсутствие старых классов в новой версии.
- **Безопасность:** на каждой паре бридж используется только если покрывает **не меньше** классов, чем obf-join; если intermediary для версии недоступен или метод хуже — автоматический откат на obf-join.
- **Отключение:** `-Dmcmigrator.intermediaryBridge=false`.

---

## Пост-проверка результата

После сборки выходной jar проверяется ASM-обходом против целевой версии: все ссылки на классы/методы/поля сверяются с целевым Minecraft-jar, всем classpath и собственными классами мода (классы JDK исключаются). Нерезолвящиеся ссылки попадают в отчёт как места ручной правки, а в «Сводку» — процент совместимости.

---

## GUI

Запуск без аргументов или с одним `.jar` («Открыть с помощью»).

### Возможности интерфейса

- **Drag & Drop** — перетащите .jar (или папку → пакетный режим) с подсветкой зоны.
- **Авто-анализ** — загрузчик, версия MC, метаданные (имя/версия/авторы определяются устойчивее, с фоллбэком из манифеста).
- **Карточка мода** — иконка из jar, бейдж загрузчика, чипы библиотек, копируемые поля, кнопка Modrinth.
- **Поля версий** — выпадающие списки + автодополнение, своп (⇄), бейдж сложности миграции, подсказка по требуемой Java, валидация.
- **Темы** — светлая / тёмная / **авто** (по системе), с запоминанием.
- **Живая смена языка** (RU/EN) **без перезапуска**.
- **Масштаб** — кнопки A−/A+ и Ctrl+колесо, с запоминанием.
- **Меню + горячие клавиши + палитра команд** (Ctrl+P).
- **Пошаговый прогресс** (степпер из 5 этапов) с ETA, кнопкой «Отмена», прогрессом в таскбаре и уведомлением по завершении.
- **Типизированные тосты** (успех/ошибка/инфо), память окна, недавние файлы, профили по умолчанию.
- **Встроенный редактор кода** — см. ниже.

### Вкладки

| Вкладка | Описание |
|---|---|
| **Журнал** | Цветной живой лог миграции с поиском (Ctrl+F) и автоскроллом |
| **Отчёт** | Фильтруемый отчёт + «Сводка», кликабельные ссылки `Файл.java:42`, экспорт/копирование |
| **Изменения (diff)** | Цветной unified-diff и режим «рядом» (side-by-side), сохранение `.patch` |
| **Совместимость** | Матрица совместимости, рекомендации, заметки |
| **Граф зависимостей** | Текстовый граф + представление деревом |
| **Проверка jar** | Preflight-валидация: дескрипторы, классы, исходники, вложенные jar |
| **Smoke-test** | Генерация скрипта `test-server-{version}.bat` |
| **Защита JAR** | Таблица механизмов защиты с цветными уровнями, выборочным снятием и сравнением «до/после» |

### Встроенный редактор кода

- Дерево файлов с нечётким фильтром, быстрым переходом (Ctrl+P), пометками правок/ошибок.
- Вкладки с preview-режимом (одиночный клик — временная, двойной — закрепить), MRU (Ctrl+Tab).
- **Автодополнение Java** (Ctrl+Space) и подсветка ошибок парсером (`languagesupport`).
- **Мини-карта** с рамкой видимой области, **split-view**, подсветка текущей строки, направляющие отступов, тема в такт окну.
- Навигация: переход к определению (Ctrl+клик), поиск использований (Ctrl+Shift+клик), структура (Ctrl+Shift+O), peek (Ctrl+Q).
- Панель **«Проблемы»** с переходом по ошибкам компиляции, перекомпиляция обратно в jar.

---

## CLI — все параметры

| Параметр | Описание |
|---|---|
| `-i, --input` | Входной `.jar` (обязательно) |
| `-o, --output` | Выходной `.jar` (обязательно) |
| `--from` | Исходная версия MC, напр. `1.20.3` (обязательно) |
| `--to` | Целевая версия MC, напр. `1.21.1` (обязательно) |
| `--type` | `fabric` / `forge` / `neoforge` / `paper` / `spigot` / `quilt` (по умолчанию: `paper`) |
| `--decompiler` | `VINEFLOWER` / `CFR` / `PROCYON` (по умолчанию: `VINEFLOWER`) |
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

### Системные свойства (`-D…`)

| Свойство | Описание |
|---|---|
| `-Dmcmigrator.intermediaryBridge=false` | Отключить intermediary-бридж, использовать только obf-join |
| `-Dmcmigrator.lang=en` | Принудительный язык интерфейса (`ru` / `en`) |

---

## Поддерживаемые версии

### Как источник (from)

Минимальная версия: **1.14.4**. Версии до 1.14.4 (1.6.4–1.14.3) отображаются в интерфейсе, но миграция с них заблокирована — нет официальных MojMap, иная экосистема модов.

### Как цель (to)

42 версии: от **1.14.4** до **26.1.2**.

### Версии Java

| MC версия | Мин. Java | Примечание |
|---|---|---|
| 1.14.4 – 1.20.4 | Java 17 | Стандарт |
| 1.20.5 – 1.21.x | Java 21 | Data components |
| 26.1+ | Java 26 | Year.drop нумерация |

Рекомпиляция всегда таргетит Java **целевой** версии MC (через `--release`), а не системную, поэтому результат не «привязывается» к Java хоста сборки. В отчёте указывается «Целевая Java: N».

---

## JSON-правила

Правила для пар версий: `src/main/resources/rules/{from}-to-{to}.json`.
**861 набор** для 42 релизов. Генерация: `python3 tools/generate_rules.py`.

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
| **PostMigrationChecker** | ASM-верификация всех ссылок выходного jar (классы/методы/поля) против целевой версии |
| **PreflightChecker** | Предварительная проверка jar: дескрипторы, классы, исходники, вложенные jar |
| **CompatibilityAdvisor** | Матрица совместимости и рекомендации |
| **LibraryCatalog** | Анализ библиотек мода |

---

## Защита JAR

### Анализ (78 типов обнаружения)

Анализатор обнаруживает: шифрование строк, обфускацию управления потоком, переименование классов, reflection, шифрование ресурсов, anti-tamper, native код, custom classloader, InvokeDynamic-обфускацию, watermarks, удаление debug-информации, anti-debug/anti-decompiler, multi-stage loader и многие другие.

### Автоматическое снятие защиты

ProtectionRemover (реализовано удаление 52 из 78 типов): расшифровка строк, упрощение потока управления, инлайн reflection, заглушивание native-методов, удаление watermarks/anti-tamper, восстановление StackMapTable, удаление мёртвого кода, упрощение InvokeDynamic, исправление illegal/unicode имён, расшифровка int/long констант, удаление CRC/hash/integrity-проверок, anti-debug/anti-decompiler и т.д. В GUI снятие **выборочное** (галочками), с показом «до/после».

> **Важно:** инструмент предназначен для модов, на которые у вас есть право на изменение. Обход лицензионных проверок чужих модов не реализуется.

---

## Ресурсы

После миграции автоматически патчатся: `fabric.mod.json`, `plugin.yml`, `mods.toml`, `pack.mcmeta` (версии/диапазоны).

---

## Auto-classpath

Для модов Fabric/Forge/NeoForge/Quilt автоматически: скачивается ванильный server/client.jar целевой версии, деобфусцируется (MojMap или intermediary для production-модов), извлекаются библиотеки и Fabric API; всё кэшируется в `~/.mcmigrator/classpath/{version}/`.

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
```

---

## Дополнительные возможности

- **Каскадная миграция** — большие переходы через промежуточные версии.
- **Пакетный режим** — миграция всех jar из папки со сводной таблицей.
- **Интерактивный выбор правил**, **Git-интеграция**, **онлайн/community-правила**.
- **Data pack / Asset правила** — рецепты, теги, лут-таблицы, модели блоков, атласы, шрифты.
- **Architectury** — детекция `@ExpectPlatform` / `@PlatformOnly`.
- **Modrinth API** — автоскачивание Fabric API и зависимостей.
- **Экспорт отчётов** (TXT/HTML/Markdown), **визуальный diff jar**, **экспорт патчей**.
- **I18n** — русский и английский интерфейс с переключением на лету.

---

## Структура проекта

```
src/main/java/com/mcmigrator/
├── analysis/          # MixinAnalyzer, DependencyChecker, ComplexityAnalyzer, PostMigrationChecker,
│                      # PreflightChecker, CompatibilityAdvisor, LibraryCatalog, MixinPortSuggester
├── classpath/         # ClasspathProvider, ModrinthResolver, AccessWidenerApplier
├── decompiler/        # DecompilerProvider, DecompilerType, ParallelDecompiler, DecompilerPluginLoader
├── gui/               # MigratorGui, ModAnalyzer, SourceEditorPanel, CodeBrowserWindow, JarRecompiler,
│                      # PipelineStepBar, UiKit, SystemTheme
├── mappings/          # MappingsProvider, MappingsDownloader, MappingChain (obf-join + intermediary-бридж),
│                      # AutoRuleGenerator
├── pipeline/          # MigrationPipeline, RemapStage, BytecodeTransformStage, DecompileStage,
│                      # TransformStage, CompileStage, CascadeMigration, BatchMigration, MigrationConfig
├── protection/        # ProtectionAnalyzer, ProtectionRemover, ProtectionFeatureScanner
├── report/            # MigrationReport, HtmlReportExporter
├── rules/             # VersionRuleSet, RulePluginLoader, OnlineRulesHub, CommunityRulesHub, …
├── transform/         # TransformRule, MethodRenameRule, ClassRenameRule, PackageChangeRule, …
├── util/              # JarUtil, GitIntegrator, Telemetry, ResourcePatcher, MinecraftVersionSupport,
│                      # UiPrefs, I18n, MigrationHistory, SettingsProfiles, DiffUtil, JarDiffUtil, …
├── Constants.java
├── Main.java
└── ModType.java
```

---

## Ограничения

- **Цепочка маппингов** — intermediary-бридж покрывает большинство классов; на очень больших скачках часть старых классов отсутствует в целевой версии и не сопоставляется.
- **Auto-classpath** — для сложных модов добавляйте зависимости через `--classpath`.
- **MojMap** — лицензия Mojang: учитывайте условия при распространении результатов.
- **Разрешение типов** в AST-правилах носит эвристический характер; неподтверждённые места помечаются в отчёте.
- **Смена загрузчика** (Forge ↔ Fabric) не поддерживается — меняется версия игры, не лоадер.
- Защищённые/обфусцированные моды могут не декомпилироваться/собираться; protection-сканер предупреждает заранее.

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
| JavaParser (symbol-solver) | 3.28.2 | AST-трансформации |
| Gson | 2.14.0 | JSON парсинг |
| SLF4J + Logback | 2.0.18 / 1.5.34 | Логирование |
| FlatLaf | 3.7.1 | GUI темы |
| RSyntaxTextArea | 3.6.2 | Подсветка синтаксиса |
| AutoComplete | 3.3.1 | Автодополнение в редакторе |
| LanguageSupport | 3.3.0 | Java-парсер для подсказок/ошибок |
| ASM | 9.9.1 | Байткод трансформации |
| JUnit 5 | 6.1.0 | Тесты |

---

## Лицензия

**MIT License** — см. [LICENSE](LICENSE).

## Ответственное использование

Инструмент предназначен для миграции модов/плагинов, **на которые у вас есть право на изменение**: собственные проекты или моды с лицензией, допускающей модификацию. Перед миграцией чужого мода:

- Проверьте лицензию мода.
- По возможности спросите автора или используйте официальную сборку для целевой версии.
- Сохраняйте атрибуты: не удаляйте имена авторов, кредиты, лицензии из результата.
