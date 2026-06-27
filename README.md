# MinecraftJarMigrator

Инструмент для автоматической миграции jar-файлов модов и плагинов Minecraft между версиями.

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
| **2. Remap** | Tiny Remapper — переименование классов/методов/полей через цепочку `named_old → obf → named_new` (MojMap) |
| **3. Decompile** | Декомпиляция байткода в Java-исходники (Vineflower / CFR / Procyon) |
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
- **Выбор декомпилятора** — Vineflower (рекомендуется), CFR, Procyon
- **Кэш декомпиляции** — SHA-256 кэш в `~/.mcmigrator/decompile-cache/`
- **Прогресс-бары** — поэтапный прогресс для каждого этапа пайплайна
- **Темы** — светлая и тёмная (FlatLaf), переключение кнопкой
- **Масштаб** — кнопки A−/A+ для изменения размера шрифта
- **Профили настроек** — сохранение/загрузка конфигураций миграции
- **История миграций** — журнал всех запусков с возможностью повтора
- **Встроенный редактор кода** — просмотр и правка декомпилированных исходников с подсветкой синтаксиса (RSyntaxTextArea), перекомпиляция в jar

### Вкладки

| Вкладка | Описание |
|---|---|
| **Журнал** | Живой лог миграции с кнопкой копирования |
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
| **Анализ защиты** | Сканирование JAR на 78 типов обфускации и защиты |
| **Снять защиту** | Автоматическое удаление обнаруженных защитных механизмов (реализованно удаление 52 типов из 78)|
| **Smoke-test** | Генерация тестового скрипта для Fabric-сервера |
| **Пакетно** | Пакетная миграция всех jar из папки |
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

---

## Поддерживаемые версии

### Как источник (from)

Минимальная версия: **1.14.4**. Версии до 1.14.4 (1.6.4–1.14.3) отображаются в интерфейсе, но миграция с них заблокирована — нет официальных MojMap, иная экосистема модов.

### Как цель (to)

42 версии: от **1.14.4** до **26.1.2**

### Версии Java

| MC версия | Мин. Java | Примечание |
|---|---|---|
| 1.14.4 – 1.20.4 | Java 17 | Стандарт |
| 1.20.5 – 1.21.x | Java 21 | Data components |
| 26.1+ | Java 26 | Year.drop нумерация |

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
| **PostMigrationChecker** | ASM-верификация всех ссылок на классы/методы/поля |
| **PreflightChecker** | Предварительная проверка jar: дескрипторы, классы, исходники, вложенные jar |
| **CompatibilityAdvisor** | Матрица совместимости и рекомендации |
| **LibraryCatalog** | Анализ библиотек мода |

---

## Защита JAR

### Анализ (78 типов обнаружения)

Анализатор обнаруживает: шифрование строк, обфускация управления потоком, переименование классов, reflection, шифрование ресурсов, anti-tamper, native код, custom classloader, ZLIB-упаковка, InvokeDynamic обфускация, обфускация чисел, watermarks, удаление debug-информации, illegal-name обфускация, control flow flattening, dead code, CRC/hash проверки, anti-debug, anti-decompiler, шифрование классов, AES-шифрование, multi-stage loader, и многие другие.

### Автоматическое снятие защиты

ProtectionRemover выполняет:
- Расшифровку строк
- Упрощение потока управления
- Инлайн reflection
- Заглушивание native методов
- Удаление watermarks и anti-tamper
- Восстановление StackMapTable
- Удаление мёртвого кода
- Упрощение InvokeDynamic
- Исправление illegal/unicode имён
- Расшифровку int/long констант
- Упрощение подмены инструкций
- Сжатие раздутых методов
- Очистку synthetic/bridge
- Исправление номеров строк
- Удаление CRC/hash/integrity проверок
- Удаление anti-debug/anti-decompiler
- Исправление CFG distortion 
 и тд. 

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

- **Каскадная миграция** — большие переходы идут через промежуточные версии
- **Пакетный режим** — миграция всех jar из папки со сводной таблицей
- **Интерактивный выбор правил** — диалог включения/отключения breaking-проверок
- **Git интеграция** — автокоммит на ветке `migrate-{from}-to-{to}`
- **Онлайн-правила** — загрузка свежих правил с GitHub без пересборки
- **Community правила** — система рейтингов (полезно/вредно)
- **Data pack правила** — трансформации рецептов, тегов, таблиц лута
- **Asset правила** — трансформации `sounds.json`, моделей блоков, атласов, шрифтов
- **Architectury поддержка** — детекция `@ExpectPlatform`, `@PlatformOnly`
- **Modrinth API** — автоскачивание Fabric API и зависимостей по ID мода
- **Телеметрия** — opt-in анонимная статистика миграций
- **Экспорт отчётов** — TXT, HTML, Markdown
- **Smoke-test** — генерация скрипта для проверки мода на Fabric-сервере
- **Визуальный diff jar** — сравнение двух jar (добавленные/удалённые/изменённые классы)
- **Экспорт патчей** — генерация `.patch` файлов из diff миграции
- **I18n** — русский и английский интерфейс

---

## Структура проекта

```
src/main/java/com/mcmigrator/
├── analysis/          # MixinAnalyzer, DependencyChecker, ComplexityAnalyzer, PostMigrationChecker,
│                      # PreflightChecker, CompatibilityAdvisor, LibraryCatalog, MixinPortSuggester
├── classpath/         # ClasspathProvider, ModrinthResolver, AccessWidenerApplier
├── decompiler/        # DecompilerProvider, DecompilerType, ParallelDecompiler, DecompilerPluginLoader
├── exception/         # MigrationException, RemapException, DecompileException, CompileException и др.
├── gui/               # MigratorGui, ModAnalyzer, SourceEditorPanel, CodeBrowserWindow, JarRecompiler
├── mappings/          # MappingsProvider, MappingsDownloader, MappingChain, AutoRuleGenerator
├── pipeline/          # MigrationPipeline, RemapStage, BytecodeTransformStage, DecompileStage,
│                      # TransformStage, CompileStage, CascadeMigration, BatchMigration, MigrationConfig
├── protection/        # ProtectionAnalyzer, ProtectionRemover, ProtectionFeatureScanner
├── report/            # MigrationReport, HtmlReportExporter
├── rules/             # VersionRuleSet, RulePluginLoader, OnlineRulesHub, CommunityRulesHub,
│                      # InteractiveRuleSelector, RuleUpdater
├── transform/         # TransformRule, MethodRenameRule, ClassRenameRule, PackageChangeRule,
│                      # ApiBreakingChangeRule, ArchitecturyRule, bytecode/, resources/
├── util/              # JarUtil, GitIntegrator, Telemetry, ResourcePatcher, FabricMetadataPatcher,
│                      # SmokeTestRunner, TestServerScriptGenerator, DiffUtil, JarDiffUtil,
│                      # MigrationHistory, SettingsProfiles, MinecraftVersionSupport, I18n и др.
├── Constants.java
├── Main.java
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
| Picocli | 4.7.5 | CLI парсинг аргументов |
| Vineflower | 1.10.1 | Декомпилятор (основной) |
| CFR | 0.152 | Декомпилятор (альтернативный) |
| Procyon | 0.6.0 | Декомпилятор (альтернативный) |
| Tiny Remapper | 0.10.3 | Ремаппинг байткода |
| Mapping IO | 0.6.1 | Чтение/запись маппингов |
| JavaParser | 3.25.8 | AST-трансформации |
| Gson | 2.10.1 | JSON парсинг |
| SLF4J + Logback | 2.0.12 / 1.5.3 | Логирование |
| FlatLaf | 3.4 | GUI темы |
| RSyntaxTextArea | 3.4.1 | Подсветка синтаксиса |
| ASM | 9.7 | Байткод трансформации |
| JUnit 5 | 5.10.2 | Тесты |

---

## Лицензия

**MIT License** — см. [LICENSE](LICENSE).

## Ответственное использование

Инструмент предназначен для миграции модов/плагинов, **на которые у вас есть право на изменение**: собственные проекты или моды с лицензией, допускающей модификацию. Перед миграцией чужого мода:

- Проверьте лицензию мода
- Предпочитаю спросить автора или использовать официальную сборку для целевой версии
- Сохраняйте атрибуты: не удаляйте имена авторов, кредиты, лицензии из результата
