# NoteBuns Launcher

**Version 5.0.2** · Desktop game launcher for the NoteBuns Minecraft server
Publisher: MilkyDev · Homepage: https://inflexus.world

---

# English

## 1. Product overview

NoteBuns Launcher is a desktop client that installs, updates and starts a modded
Minecraft installation for a single private game server. It is distributed as a
portable executable and is used exclusively by players of that server.

The launcher performs four jobs:

1. Keeps the local modpack identical to the version published by the server.
2. Provisions and validates a private Java runtime for the game.
3. Starts the game with the correct parameters and monitors the session.
4. Verifies client integrity to keep multiplayer fair.

The application is not advertising-supported, does not bundle third-party
software, does not modify system settings, and does not persist outside its own
installation directory and the per-user application data folder.

## 2. Technology

| Layer | Technology |
|---|---|
| User interface | React 19, TypeScript, Vite, Tailwind CSS |
| Desktop runtime | Tauri 2 (system WebView; no bundled browser engine) |
| Application core | Rust (edition 2021), Tokio, Reqwest |
| Integrity | SHA-256, BLAKE3, Ed25519 |

## 3. Component responsibilities

| Component | Responsibility |
|---|---|
| `minecraft/sync.rs` | Delta synchronisation of the modpack against the server index; SHA-256 comparison; atomic downloads |
| `minecraft/launch.rs` | Assembles the Java command line, classpath and JVM parameters; starts the game |
| `minecraft/anticheat.rs` | Client integrity verification before and during a session |
| `minecraft/whitelist.rs` | Retrieval and Ed25519 signature verification of the exemption list |
| `minecraft/migrate.rs`, `relocate.rs` | Migration of legacy installations; moving the game directory between drives |
| `java/download.rs` | Provisioning and validation of the Temurin Java runtime |
| `launcher/config.rs` | Reading and atomic writing of user settings |
| `utils/crash_log_handler.rs` | Session events, crash diagnostics and abuse-prevention reporting |
| `utils/hash.rs` | SHA-1, SHA-256 and BLAKE3 file hashing |

## 4. Behaviours that security software may flag

This section documents every operation that heuristic engines commonly treat as
suspicious, the reason it exists, and the constraints applied to it. All of it is
observable in the published source.

### 4.1 Process enumeration

**Observed behaviour.** Every five seconds during an active game session the
launcher enumerates running process names.

**Purpose.** Detection of memory editors and injection tools used to cheat in
multiplayer (for example Cheat Engine, debuggers, known injector utilities).

**Constraints.** Only process *names* are requested. The launcher does not read
command lines, environment blocks, memory, loaded modules or handles of any
process. Enumeration stops when no game session is running.

### 4.2 Process termination

**Observed behaviour.** The launcher can terminate a process tree.

**Purpose.** Ending the game session when a violation is detected, and stopping
the game when the user closes the launcher.

**Constraints.** Termination is limited to the Java process the launcher itself
started and its descendants. The target is resolved by walking the parent-child
tree downward from the launcher's own child process, so unrelated processes
cannot be reached. Termination uses the native operating-system API; the launcher
does not invoke `taskkill`, `kill`, `pkill` or any external utility.

### 4.3 Replacement of the launcher executable

**Observed behaviour.** The launcher can download a new copy of itself and
replace the running executable.

**Purpose.** Delivering updates without requiring a manual reinstall.

**Constraints.** The update is downloaded exclusively from the project's GitHub
releases over TLS. When the release publishes a checksum sidecar, the SHA-256 of
the downloaded file is verified before installation and a mismatch aborts the
update. Installation is performed by renaming files within the installation
directory — no temporary scripts are written and no command interpreter is
invoked. The previous executable is retained and removed on the next start. A
failed swap is rolled back automatically. Updates are only ever applied when the
published version is strictly newer than the installed one.

### 4.4 Downloading and executing a Java runtime

**Observed behaviour.** The launcher downloads an archive, extracts it, and
executes a binary from it.

**Purpose.** Minecraft requires a Java runtime. The launcher provisions a private
copy rather than depending on a system-wide installation, so the game always runs
on a known, tested version.

**Constraints.** The runtime is Eclipse Temurin, downloaded over TLS from the
official Adoptium distribution. Its SHA-256 is verified against a value compiled
into the launcher before extraction. Archive extraction rejects absolute paths
and directory traversal, so entries cannot be written outside the target
directory. The runtime is installed inside the game directory and is used only to
start Minecraft.

### 4.5 Encrypted outbound network traffic

**Observed behaviour.** The launcher sends encrypted payloads to
`handler.milkycloud.online` over TLS.

**Purpose.** Session accounting, crash diagnostics and abuse prevention. See
section 6.

**Constraints.** The transport is HTTPS. The request body is JSON with named
fields that identify the product, its version, and the protection algorithms in
use (RSA-OAEP-SHA256 for the key, AES-256-GCM for the content); only the payload
itself is encrypted. The request carries a User-Agent naming the product,
version and homepage. Encryption protects player data in transit; it is not used
to conceal the protocol, whose structure is documented here and in the source.

### 4.6 Hardware fingerprint

**Observed behaviour.** The launcher derives a stable identifier from machine
characteristics.

**Purpose.** Correlating a sanctioned account with the machine it used, so that a
banned cheater cannot immediately return under a new nickname.

**Constraints.** The identifier is a truncated SHA-256 digest of the operating
system name and version, CPU architecture, the local account name, the CPU model
string and the total installed memory. It is a one-way hash; the source values
cannot be recovered from it. No serial numbers, disk identifiers, MAC addresses
or hardware IDs are read.

## 5. Client integrity verification

The launcher enforces the published modpack. The mechanism is entirely local: the
rules are loaded once at start-up and evaluated in memory, with no network
request per check.

**Reference set.** At start-up the launcher loads the server's file index, which
lists every legitimate file with its expected size and SHA-256.

**Start-up verification.** Before the game launches, the contents of
`resourcepacks/` and `texturepacks/` are compared against the reference set.
Unrecognised files are removed. If a file cannot be removed, the launch is
refused.

**Session verification.** While the game runs, `mods/`, `resourcepacks/` and
`texturepacks/` are re-examined every five seconds. A file that is absent from
the reference set is removed and the session is ended. A file that is present but
whose content no longer matches the expected size and hash is treated the same
way — this detects a legitimate file whose contents were swapped after launch.
Hashes are recomputed only for files whose modification time changed, so
untouched files impose no disk cost.

**Resource pack whitelist.** Resource packs are matched by SHA-256 against a list
published by the server. This blocks packs that grant an unfair advantage, such
as X-Ray textures or transparent blocks, while allowing players to request that a
cosmetic pack be added to the permitted list.

**Exemption list integrity.** The list of exempt nicknames and permitted pack
hashes is signed with Ed25519 and verified against a public key compiled into the
launcher. A list that fails verification, has expired, or cannot be retrieved is
discarded and full enforcement remains in effect. Consequently the list cannot be
forged locally to disable verification, and blocking the server does not weaken
it.

**Scope.** Verification is confined to the game directory. The launcher does not
inspect, modify or report on unrelated files, applications or user documents.

## 6. Telemetry and abuse prevention

Multiplayer integrity requires that sanctions be enforceable. Without a durable
signal, a banned account is replaced within minutes and the sanction has no
effect. Telemetry exists to make enforcement possible; it is not used for
advertising, profiling or resale, and no data is shared with third parties.

**Events reported.** Launcher start, game session start, game session end,
launcher exit, integrity violation, and game crash.

**Data transmitted.** Player nickname, the derived hardware fingerprint, the
network interface address, operating system name and version, CPU model and core
count, total and allocated memory, launcher version, and timestamps. Violation
reports additionally name the offending file or process and the reason. Crash
reports include the tail of the game log, and only when the user has left crash
reporting enabled in settings.

**Not collected.** Browsing history, credentials, documents, keystrokes,
screenshots, clipboard contents, contacts, location, and the contents of any file
outside the game directory.

**User control.** Crash report submission is a toggle in the launcher settings.

## 7. Build and verification

The application is built by a public GitHub Actions workflow from the source in
this repository. Release binaries are published as GitHub release assets with
SHA-256 checksums.

```bash
npm install
```

```bash
npm run build
```

```bash
cargo test --manifest-path src-tauri/Cargo.toml
```

```bash
npx tauri build
```

The release profile applies ordinary optimisation only. No packer, protector,
obfuscator or anti-debugging technique is used at any stage, and the build script
performs no code generation beyond the standard Tauri step.

## 8. Reporting a false positive

If this application has been classified as malicious, we ask for a re-evaluation.
The complete source, build workflow and release artefacts are public, and the
behaviours most likely to have triggered a heuristic are documented in section 4.

Contact: https://inflexus.world

---

# Русская версия

> Раздел адресован специалистам антивирусных лабораторий и содержит сведения,
> необходимые для оценки поведения приложения.

## 1. Назначение продукта

NoteBuns Launcher — настольный клиент, который устанавливает, обновляет и
запускает модифицированную сборку Minecraft для одного частного игрового
сервера. Распространяется в виде портативного исполняемого файла и используется
исключительно игроками этого сервера.

Приложение решает четыре задачи:

1. Поддерживает локальную сборку в точном соответствии с версией на сервере.
2. Устанавливает и проверяет отдельную среду Java для игры.
3. Запускает игру с корректными параметрами и наблюдает за сессией.
4. Проверяет целостность клиента для соблюдения честной игры.

Приложение не содержит рекламы, не устанавливает стороннее программное
обеспечение, не изменяет системные настройки и не размещает файлы за пределами
собственного каталога установки и пользовательской папки данных приложения.

## 2. Технологии

| Уровень | Технологии |
|---|---|
| Интерфейс | React 19, TypeScript, Vite, Tailwind CSS |
| Среда выполнения | Tauri 2 (системный WebView, без встроенного браузерного движка) |
| Ядро приложения | Rust (edition 2021), Tokio, Reqwest |
| Целостность | SHA-256, BLAKE3, Ed25519 |

## 3. Назначение компонентов

| Компонент | Назначение |
|---|---|
| `minecraft/sync.rs` | Дельта-синхронизация сборки с серверным индексом, сверка SHA-256, атомарная загрузка |
| `minecraft/launch.rs` | Формирование командной строки Java, classpath и параметров JVM, запуск игры |
| `minecraft/anticheat.rs` | Проверка целостности клиента до и во время сессии |
| `minecraft/whitelist.rs` | Получение и проверка подписи Ed25519 у списка исключений |
| `minecraft/migrate.rs`, `relocate.rs` | Миграция старых установок, перенос каталога игры между дисками |
| `java/download.rs` | Установка и проверка среды выполнения Java Temurin |
| `launcher/config.rs` | Чтение и атомарная запись пользовательских настроек |
| `utils/crash_log_handler.rs` | События сессии, диагностика сбоев и предотвращение злоупотреблений |
| `utils/hash.rs` | Хеширование файлов SHA-1, SHA-256 и BLAKE3 |

## 4. Поведение, вызывающее срабатывание эвристик

Ниже описана каждая операция, которую эвристические механизмы традиционно
считают подозрительной, причина её наличия и наложенные ограничения. Всё
перечисленное наблюдаемо в опубликованных исходных текстах.

### 4.1 Перечисление процессов

**Наблюдаемое поведение.** Каждые пять секунд во время игровой сессии приложение
перечисляет имена запущенных процессов.

**Назначение.** Обнаружение редакторов памяти и средств внедрения кода,
применяемых для получения преимущества в многопользовательской игре.

**Ограничения.** Запрашиваются только *имена* процессов. Приложение не читает
командные строки, блоки окружения, память, загруженные модули и дескрипторы
каких-либо процессов. Вне игровой сессии перечисление не выполняется.

### 4.2 Завершение процессов

**Наблюдаемое поведение.** Приложение может завершить дерево процессов.

**Назначение.** Прекращение игровой сессии при обнаружении нарушения и остановка
игры при закрытии лаунчера.

**Ограничения.** Завершение ограничено процессом Java, который запустил сам
лаунчер, и его потомками. Цель определяется обходом дерева «родитель-потомок»
вниз от собственного дочернего процесса, поэтому посторонние процессы
недостижимы. Используется системный интерфейс операционной системы; утилиты
`taskkill`, `kill`, `pkill` и любые иные внешние программы не вызываются.

### 4.3 Замена исполняемого файла лаунчера

**Наблюдаемое поведение.** Приложение может загрузить свою новую копию и
заменить работающий исполняемый файл.

**Назначение.** Доставка обновлений без ручной переустановки.

**Ограничения.** Обновление загружается исключительно из релизов проекта на
GitHub по TLS. Если релиз публикует контрольную сумму, SHA-256 загруженного
файла проверяется до установки, и при несовпадении обновление прерывается.
Установка выполняется переименованием файлов внутри каталога установки:
временные сценарии не создаются, командный интерпретатор не запускается.
Предыдущий исполняемый файл сохраняется и удаляется при следующем запуске. При
неудачной замене выполняется автоматический откат. Обновление применяется только
если опубликованная версия строго новее установленной.

### 4.4 Загрузка и запуск среды Java

**Наблюдаемое поведение.** Приложение загружает архив, распаковывает его и
запускает содержащийся в нём исполняемый файл.

**Назначение.** Minecraft требует среды выполнения Java. Лаунчер устанавливает
отдельную копию вместо использования системной, чтобы игра всегда работала на
известной проверенной версии.

**Ограничения.** Используется Eclipse Temurin, загружаемый по TLS из
официального дистрибутива Adoptium. Его SHA-256 сверяется со значением, вшитым в
лаунчер, до распаковки. При распаковке отвергаются абсолютные пути и выход за
пределы каталога, поэтому записать файл вне целевого каталога невозможно. Среда
размещается внутри каталога игры и применяется только для запуска Minecraft.

### 4.5 Шифрованный исходящий трафик

**Наблюдаемое поведение.** Приложение отправляет шифрованные сообщения на
`handler.milkycloud.online` по TLS.

**Назначение.** Учёт сессий, диагностика сбоев и предотвращение злоупотреблений.
См. раздел 6.

**Ограничения.** Транспорт — HTTPS. Тело запроса представляет собой JSON с
именованными полями, указывающими продукт, его версию и применяемые алгоритмы
защиты (RSA-OAEP-SHA256 для ключа, AES-256-GCM для содержимого); шифруется
только полезная нагрузка. Запрос снабжён заголовком User-Agent с названием
продукта, версией и адресом сайта. Шифрование защищает данные игроков при
передаче и не служит сокрытию протокола, структура которого описана здесь и в
исходных текстах.

### 4.6 Отпечаток оборудования

**Наблюдаемое поведение.** Приложение вычисляет устойчивый идентификатор машины.

**Назначение.** Сопоставление наказанной учётной записи с использованным
устройством, чтобы заблокированный нарушитель не возвращался немедленно под
новым псевдонимом.

**Ограничения.** Идентификатор — усечённый хеш SHA-256 от названия и версии
операционной системы, архитектуры процессора, имени локальной учётной записи,
строки модели процессора и общего объёма памяти. Преобразование одностороннее,
исходные значения из него не восстанавливаются. Серийные номера, идентификаторы
дисков, MAC-адреса и аппаратные идентификаторы не считываются.

## 5. Проверка целостности клиента

Лаунчер обеспечивает соответствие опубликованной сборке. Механизм полностью
локальный: правила загружаются один раз при запуске и вычисляются в памяти, без
сетевого запроса на каждую проверку.

**Эталонный набор.** При запуске загружается серверный индекс файлов, содержащий
перечень легитимных файлов с ожидаемым размером и SHA-256.

**Проверка при запуске.** До старта игры содержимое каталогов `resourcepacks/` и
`texturepacks/` сверяется с эталонным набором. Неопознанные файлы удаляются.
Если файл удалить не удаётся, запуск запрещается.

**Проверка во время сессии.** Во время игры каталоги `mods/`, `resourcepacks/` и
`texturepacks/` перепроверяются каждые пять секунд. Файл, отсутствующий в
эталонном наборе, удаляется, а сессия прекращается. Так же обрабатывается файл,
который присутствует в наборе, но чьё содержимое перестало соответствовать
ожидаемому размеру и хешу — это выявляет подмену содержимого легитимного файла
после запуска. Хеши пересчитываются только для файлов с изменившимся временем
модификации, поэтому нетронутые файлы не создают дисковой нагрузки.

**Вайтлист ресурспаков.** Ресурспаки сверяются по SHA-256 со списком,
публикуемым сервером. Это блокирует пакеты, дающие нечестное преимущество —
текстуры X-Ray, прозрачные блоки и подобные, — оставляя игрокам возможность
запросить добавление косметического пакета в разрешённый список.

**Целостность списка исключений.** Список освобождённых псевдонимов и
разрешённых хешей подписан Ed25519 и проверяется открытым ключом, вшитым в
лаунчер. Список, не прошедший проверку, просроченный или недоступный,
отбрасывается, а проверки продолжают действовать в полном объёме. Тем самым
список нельзя подделать локально для отключения проверок, а блокировка сервера
их не ослабляет.

**Область действия.** Проверка ограничена каталогом игры. Лаунчер не
просматривает, не изменяет и не передаёт сведения о посторонних файлах,
приложениях и документах пользователя.

## 6. Телеметрия и предотвращение злоупотреблений

Честность многопользовательской игры требует исполнимости санкций. Без
устойчивого признака заблокированная учётная запись заменяется за считанные
минуты, и наказание теряет смысл. Телеметрия существует именно для этого; она не
применяется для рекламы, профилирования или перепродажи, данные третьим лицам не
передаются.

**Передаваемые события.** Запуск лаунчера, начало игровой сессии, завершение
игровой сессии, выход из лаунчера, нарушение целостности, сбой игры.

**Передаваемые данные.** Игровой псевдоним, вычисленный отпечаток оборудования,
адрес сетевого интерфейса, название и версия операционной системы, модель и
число ядер процессора, общий и выделенный объём памяти, версия лаунчера,
временные метки. Сообщения о нарушении дополнительно содержат имя файла или
процесса и причину. Отчёты о сбоях включают конец журнала игры и отправляются
только при включённой пользователем настройке.

**Не собирается.** История посещений, учётные данные, документы, нажатия клавиш,
снимки экрана, содержимое буфера обмена, контакты, местоположение и содержимое
любых файлов вне каталога игры.

**Управление.** Отправка отчётов о сбоях управляется переключателем в настройках
лаунчера.

## 7. Сборка и проверка

Приложение собирается публичным процессом GitHub Actions из исходных текстов
этого репозитория. Двоичные файлы публикуются как вложения релизов GitHub с
контрольными суммами SHA-256.

```bash
npm install
```

```bash
npm run build
```

```bash
cargo test --manifest-path src-tauri/Cargo.toml
```

```bash
npx tauri build
```

Релизный профиль применяет только обычную оптимизацию. Упаковщики, протекторы,
обфускаторы и приёмы противодействия отладке не используются ни на одном этапе,
а сборочный сценарий не выполняет генерации кода сверх стандартного шага Tauri.

## 8. Обращение по ложному срабатыванию

Если приложение классифицировано как вредоносное, мы просим о повторной оценке.
Исходные тексты, процесс сборки и артефакты релизов открыты, а поведение,
наиболее вероятно вызвавшее срабатывание эвристики, описано в разделе 4.

Контакт: https://inflexus.world
