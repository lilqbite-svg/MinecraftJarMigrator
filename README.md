![Uploading explorer_j4LrQ73plQ.png…]()
# MinecraftJarMigrator

Инструмент для миграции jar-файлов модов/плагинов Minecraft между версиями:

```
inputJar → [1.5/4] BytecodeTransform (ASM)
         → [2/4] Remap (Tiny Remapper)
         → [3/4] Decompile (Vineflower / CFR / Procyon)
         → [4/4] Transform (JavaParser + JSON rules)
         → [5/4] Compile (javax.tools) → outputJar + report
```

Mappings (official MojMap) are downloaded from Mojang servers automatically
and cached in `~/.mcmigrator/mappings/{version}-mojmap.tiny`.

## Build

Requires **JDK 17+** and internet access (Maven Central + maven.fabricmc.net).

```bash
./gradlew shadowJar
# → build/libs/migrator.jar (fat-jar with all dependencies)
```

## Windows .exe (embedded Java)

Run **on Windows with JDK 17+**:

```bat
build-exe.bat
```

**No Gradle needed**: the script downloads Gradle 9.5.1 automatically if not found.
Result: `build\dist\MinecraftJarMigrator\MinecraftJarMigrator.exe` — runs on any Windows
without Java installed. Archive the folder and distribute as-is.

## GUI

Run **without arguments** (`MinecraftJarMigrator.exe` or `java -jar migrator.jar`) to open GUI:

* **Drag & drop**: drop a .jar to start analysis automatically
* **Auto-analysis**: loader type (Fabric/Quilt/Forge/NeoForge/Paper/Spigot), MC version, metadata
* **Mod info panel**: name, ID, version, MC, loader, authors, license, description
* **Decompiler selector**: Vineflower (default), CFR, Procyon — choose in dropdown
* **Decompilation cache**: toggle on/off for faster repeated migrations
* **Per-stage progress bars**: Remap / BytecodeTransform / Decompile / Transform / Compile
* **Light/dark theme** toggle (FlatLaf)
* **Source editor** with syntax highlighting (RSyntaxTextArea)

### Tabs

| Tab | Description |
|---|---|
| Journal | Live migration log with copy button |
| Report | Filtered report (transformations, manual fixes, warnings) |
| Changes (diff) | Unified diff for each modified file |
| Compatibility | Library analysis, dependency graph |
| Dependency graph | Visual dependency tree |
| JAR check | Preflight validation |
| Smoke-test | Test server script generation |
| Protection | JAR protection analysis and removal |

## CLI

```bash
java -jar migrator.jar \
  --input my-plugin-1.20.3.jar \
  --output my-plugin-1.21.1.jar \
  --from 1.20.3 \
  --to 1.21.1 \
  --type paper
```

### Options

| Option | Description |
|---|---|
| `-i, --input` | Input `.jar` (required) |
| `-o, --output` | Output `.jar` (required) |
| `--from` | Source MC version, e.g. `1.20.3` (required) |
| `--to` | Target MC version, e.g. `1.21.1` (required) |
| `--type` | `fabric` / `forge` / `neoforge` / `paper` / `spigot` / `quilt` (default: `paper`) |
| `--decompiler` | `VINEFLOWER` / `CFR` / `PROCYON` (default: `VINEFLOWER`) |
| `--no-decompile-cache` | Disable decompilation result caching |
| `--interactive` | Interactive rule selection before migration |
| `--classpath` | Extra classpath for recompilation (comma-separated) |
| `--auto-classpath` | Download classpath automatically (default: on; disable: `--no-auto-classpath`) |
| `--keep-sources` | Keep decompiled sources in `{output}-sources/` |
| `--dry-run` | Show changes without writing files outside temp dir |
| `--cascade` | Cascade through intermediate versions |
| `--batch` | Batch mode: `--input` and `--output` are folders |
| `--disable-rule` | Disable a rule by description (repeatable) |
| `--mappings-dir` | Mappings cache dir (default: `~/.mcmigrator/mappings`) |
| `-v, --verbose` | DEBUG level output |

After a successful run, `{output}.report.txt` appears alongside the output jar.

## Pipeline Stages

### 1. Bytecode Transform (ASM)
Applies bytecode-level transformations before decompilation:
- **RefmapPatcher** — remaps intermediary names in `*-refmap.json` files
- **MixinConfigPatcher** — updates `compatibilityLevel` (JAVA_17/21/26)
- **AccessWidenerPatcher** — remaps names in `.accesswidener` files

### 2. Remap (Tiny Remapper)
Renames classes/methods/fields from source version MojMap to target version MojMap
via the chain `named_old → obf → named_new`.

### 3. Decompile
Converts bytecode to Java sources. Supports multiple backends:
- **Vineflower** (default, recommended)
- **CFR** (optional, add `cfr-decompiler` dependency)
- **Procyon** (optional, add `procyon-decompiler` dependency)

Fallback mode: if Vineflower fails, tries CFR, then Procyon automatically.
Parallel decompilation: processes classes in chunks across CPU cores.

### 4. Transform (JavaParser + JSON rules)
Applies AST-level transformations from `src/main/resources/rules/{from}-to-{to}.json`:
- `methodRenames` — automatic method renames
- `classRenames` — automatic class renames
- `packageMoves` — automatic package relocations
- `apiBreakingChanges` — report-only markers for manual fixes

### 5. Compile (javax.tools)
Recompiles modified sources. Per-file fallback: if batch compilation fails,
each file is compiled individually — successes go into the jar, failures
revert to remapped bytecode with detailed error reporting.

## JSON Rules

Rules for version pairs are in: `src/main/resources/rules/{from}-to-{to}.json`.
**861 pairs generated** for 42 releases from 1.14.4 to 26.1.2.
Regenerate: `python3 tools/generate_rules.py`.

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

## Features (Full List)

### Core Pipeline
1. **Cascade migration** — checkbox "Cascade" (CLI `--cascade`): large jumps go through intermediate versions
2. **Batch mode** — button "Batch" (CLI `--batch`): migrate all jars from a folder with summary table
3. **Mixin analysis** — button "Check Mixins": verifies mixin targets against target version jar
4. **Drag & Drop + auto-analysis** — drop jar: loader type, MC version, metadata auto-detected
5. **Diff viewer** — tab "Changes": unified diff for each rule-modified file
6. **Plugin system** — custom `TransformRule` jars from `~/.mcmigrator/plugins/`
7. **Online rule updates** — fresh rules from GitHub without rebuilding (cache in `~/.mcmigrator/rules`)
8. **Mixin port hints** — for broken targets, candidates are suggested by signature
9. **Forge/NeoForge/Quilt classpath** — auto-classpath downloads available artifacts
10. **Test server script** — `test-server-<version>.bat` generated alongside output jar
11. **Interactive breaking rules** — dialog to enable/disable individual API checks before migration
12. **Report tab with filter, history, profiles** — report in-window, migration history, saveable profiles; export to HTML/Markdown

### Decompilation
13. **Multi-decompiler** — Vineflower / CFR / Procyon selector in GUI and CLI
14. **Decompiler fallback** — if primary fails, automatically tries next backend
15. **Parallel decompilation** — classes processed in chunks across CPU cores
16. **Decompilation cache** — SHA-256 hash-based cache in `~/.mcmigrator/decompile-cache/`

### Bytecode & Protection
17. **ASM bytecode transforms** — RefmapPatcher, MixinConfigPatcher, AccessWidenerPatcher
18. **Per-file compilation fallback** — individual file compilation on batch failure
19. **Resource file patching** — `fabric.mod.json`, `plugin.yml`, `mods.toml`, `pack.mcmeta`
20. **JAR protection analysis** — detects 14 protection types (string encryption, control flow obfuscation, etc.)
21. **JAR protection removal** — automatic removal of detected protections (string decryption, anti-tamper, etc.)

### Analysis
22. **Dependency checking** — validates `fabric.mod.json`, `mods.toml`, `plugin.yml` compatibility
23. **Complexity analyzer** — pre-migration score (TRIVIAL → MANUAL) with risk assessment
24. **Post-migration checker** — ASM-level verification of all class/method/field references
25. **Auto Mixin porting** — field suggestions + auto-portable detection (single-candidate matches)
26. **Auto-generated rules from MojMap diffs** — generates renames by comparing two mapping trees
27. **Modrinth API integration** — auto-download Fabric API and dependencies by mod ID
28. **Architectury support** — `@ExpectPlatform`, `@PlatformOnly` detection

### Infrastructure
29. **Git integration** — auto-commit on branch `migrate-{from}-to-{to}`, diff, reset
30. **Online rules hub** — GitHub-based rule fetching with SHA caching
31. **Community rules hub** — rating system (helpful/harmful) for rules
32. **Telemetry** — opt-in anonymous migration statistics
33. **Source editor** — RSyntaxTextArea with Java/JSON/YAML syntax highlighting
34. **Visual JAR diff** — compare two jars (added/removed/modified classes)
35. **Patch export** — generate `.patch` files from migration diff
36. **Data pack rules** — recipe, tag, loot table transformations for version changes
37. **Asset rules** — `sounds.json`, block models, atlas, font transformations
38. **Migration profiles** — saveable settings (decompiler, type, classpath, flags)
39. **GitHub Action** — CI/CD migration in `.github/action.yml`
40. **Java 26 support** — `--release 26`, `JAVA_26` compatibility level, auto-detection

## Java Version Support

| MC Version | Min Java | Notes |
|---|---|---|
| 1.14.4 – 1.20.4 | Java 17 | Standard |
| 1.20.5 – 1.21.x | Java 21 | Data components, new features |
| 26.1+ | Java 26 | Year.drop numbering, full deobfuscation |

## Important Limitations

* **Mapping chain** is `named_old → obf → named_new` — only matching obf names are covered
* **Auto-classpath** downloads vanilla server.jar + libraries; for complex mods add via `--classpath`
* **MojMap** are under Mojang license — consider its terms when distributing results
* **JavaParser pretty-printer** reformats modified files — original formatting is not preserved
* **Type resolution** in MethodRenameRule is best-effort — unconfirmed types are marked in report
* **Mappings download** requires access to `launchermeta.mojang.com` and `piston-data.mojang.com`

## Project Structure

```
src/main/java/com/mcmigrator/
├── analysis/          # MixinAnalyzer, DependencyChecker, ComplexityAnalyzer, PostMigrationChecker
├── classpath/         # ClasspathProvider, ModrinthResolver
├── decompiler/        # DecompilerProvider, DecompilerType, ParallelDecompiler, DecompilerPluginLoader
├── exception/         # MigrationException hierarchy
├── gui/               # MigratorGui, ModAnalyzer, SourceEditorPanel
├── mappings/          # MappingsProvider, MappingsDownloader, MappingChain, AutoRuleGenerator
├── pipeline/          # MigrationPipeline, stages, config, context
├── protection/        # ProtectionAnalyzer, ProtectionRemover
├── report/            # MigrationReport, HtmlReportExporter
├── rules/             # VersionRuleSet, RulePluginLoader, OnlineRulesHub, CommunityRulesHub, InteractiveRuleSelector
├── transform/         # TransformRule, bytecode/, resources/, ArchitecturyRule
├── util/              # JarUtil, GitIntegrator, Telemetry, ResourcePatcher, SmokeTestRunner, etc.
├── Constants.java
├── Main.java
└── ModType.java
```

## License

See LICENSE file.
