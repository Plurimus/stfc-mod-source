# stfc-mod-source

[English](#english) | [Русский](#русский)

---

## English

A mod source for STFC Borg.Box. The only required file is `catalog.json` at the root of this
repository.

### Source link

Always use the direct (raw) link, not the GitHub file-viewer page:

```
https://raw.githubusercontent.com/Plurimus/stfc-mod-source/refs/heads/main/catalog.json
```

This link goes in two places:
1. In Borg.Box → central hub settings → "Mod sources" → "Add by link".
2. In Borg.Box → assembly module → "Mod source link" field — when building **each** of your mods
   (this is a pointer to where trust gets verified, not to where any particular file was
   downloaded from).

Borg.Box also requires at least **one local folder** mod source (central hub settings → "Mod
sources" → "Local folder") — the app downloads a copy of a mod into that folder before installing
it, and validates it from there (see "Three-level validation" below). A source-only catalog link
by itself is not enough to install anything.

### catalog.json structure

- `authors[]` — trusted authors. `publicKeys[]` is a list of the author's public keys (see key
  rotation below), each one obtained from the "Generate new key" button in Borg.Box's assembly
  module. Only the **public** key's `pem` goes into `catalog.json` - never the private one.
- `mods[]` — one entry per mod **version** (Borg.Box shows one node per version, chained into a
  branch per mod id). Fields:
  - `archive` — an absolute or catalog-relative link to a **single** `*.mod` file (the manifest
    already lives inside it - Borg.Box never produces a separate manifest file). The `.mod`
    itself doesn't need to live in this repo - keeping it in the mod's own repo release assets
    works fine.
  - `sha256`/`signature` — see "Publishing your own mod" below.
  - `icon` — base64-encoded SVG (same bbox-fit-into-a-circle convention as the assembly module's
    icon field, see "Icon template" below). Recommended: lets Borg.Box show the real icon on a
    not-yet-installed mod's tree node without downloading the whole archive. Falls back to a
    generic placeholder icon if omitted.
  - `shortestDescription` — recommended: shown in the node's tooltip/panel. Falls back to
    `summary` if omitted.
  - `descriptions` — optional per-language overrides, e.g. `{"ru": {"summary": "...",
    "shortestDescription": "..."}}`. Borg.Box uses the current UI language's entry when present,
    otherwise falls back to the mod's own (language-neutral default) `summary`/
    `shortestDescription`. Add whichever language codes you have real translations for - none of
    this is required.

### Mod archive (`.mod`) structure

A `.mod` file is a plain zip archive (see "Archive trailer" below for the extra bytes appended
after it) with `manifest.json` at its root, plus every file that manifest refers to, each at
exactly the path named in the manifest:

```
manifest.json
Your.Plugin.dll           # referenced by installFiles[].source
README.md                 # referenced by instructionsFile (optional)
changelog.md              # referenced by changelogFile (optional)
Screenshot1.png           # referenced by screenshots[] (optional)
```

`manifest.json` fields:

- `id`, `name`, `version`, `authorId` — identity; `id` is the stable key Borg.Box uses for install
  state and update comparisons, so keep it constant across versions of the same mod.
- `type` — `0` for a BepInEx plugin (installs into `BepInEx/plugins`), `1` for a community patch
  (installs into the client root).
- `summary`, `shortestDescription` — see `catalog.json structure` above (same fields, kept in sync
  between the manifest and the catalog entry).
- `instructionsFile` — path (inside this archive) to a Markdown file with the full description/
  usage instructions, shown in Borg.Box's README tab after a successful download. Optional.
- `changelogFile` — path (inside this archive) to a **separate** Markdown file listing what changed
  in this version, shown in its own Changelog tab next to README (added alongside `instructionsFile`
  — a mod doesn't need one just because it has the other). Optional.
- `icon` / `iconSvgBase64` — prefer `iconSvgBase64` (same base64 SVG as `catalog.json`'s `icon`
  field, embedded directly rather than as an archive path) — `icon` is a legacy path-based
  fallback.
- `screenshots[]` — paths (inside this archive) to screenshot images. Optional.
- `installFiles[]` — `{"source": "<path in this archive>", "target": "<path under the game's
  plugins/root folder>"}` for every file this mod actually installs.
- `minGameVersion` — optional lower bound on the game client version.
- `fileHashes[]` — SHA-256 of every file in the archive except `manifest.json` itself (covers
  `installFiles`, `instructionsFile`, `changelogFile`, `screenshots`, `icon`), computed by the
  signing tool at packing time — see "Three-level validation" below.
- `signature` — see "Publishing your own mod" below.

### Three-level validation

Borg.Box checks a mod at three points, so a bad or tampered file is caught as early as possible:

1. **Before download** (url sources): straight from `catalog.json` itself - id, name, version,
   description, icon, `sha256`, `signature`, author's public keys.
2. **After download, before unpacking**: once the download is cached in the local mod source
   folder, Borg.Box reads it back from there and checks name/version/author/sourceRef against the
   archive's own **trailer** (see below) - a cheap check with no unzip needed.
3. **After unpacking**: the manifest-level signature inside `manifest.json` is verified against
   the author's public key(s) (see key rotation below), and every `fileHashes` entry is checked
   against the actual extracted bytes.

### Archive trailer (MODTRL03)

Every `.mod` built by Borg.Box carries a small binary trailer after the zip content, primarily so
step 2 above can happen without unzipping: magic bytes (`MODTRL03`), `authorId`, `sourceRef`,
`name`, `version`, `shortestDescription`, `icon` (base64 SVG) and the archive-level signature -
all mirrored from `manifest.json`/`catalog.json`, not a separate source of truth. A trailer is
always present, even on an unsigned build (empty signature), so any tool can tell "this is a
Borg.Box mod" from the last few bytes alone, without touching the zip.

### Multiple signatures per author (key rotation)

An author can have **more than one** key listed in `publicKeys[]` at the same time - this covers
the case where a private key is lost (disk failure, key not backed up, etc.) but access to the
source repository itself is retained. In that case:

1. Generate a NEW key in Borg.Box ("Generate new key").
2. Add it as ANOTHER entry in the same author's `publicKeys[]` - do **not** remove the old one(s).
   Mark the new one `"status": "active"`, and change the old one(s) to `"status": "revoked"`
   (`revoked` is purely informational - "not used for new signatures anymore" - it does NOT block
   verification of older mods).
3. Sign all NEW mods with the new key.
4. Mods signed with the previous key stay verifiable: verification must try **every** key listed
   for that author, in turn (regardless of `status`), until one matches - so older plugins don't
   lose their trusted status just because the key rotated.

Keys in `publicKeys[]` are never permanently deleted (removing one would stop already-published mod
versions from verifying) - just marked `revoked` if you want to visibly record that the key changed.

### Publishing your own mod

1. Build and sign the mod in Borg.Box ("Собрать мод"/"Build mod" button, with a signing key
   loaded and the source link filled in - see above). The output is a single file:
   `<id>-<version>.mod`.
2. Upload that file as a release asset in the mod's own repository, e.g.:
   ```
   https://github.com/<author>/<mod-repo>/releases/download/v1.0.0/<name>-1.0.0.mod
   ```
3. Compute the SHA-256 of the `*.mod` file (e.g. `certutil -hashfile file.mod SHA256` on Windows).
4. Add/update the entry in this `catalog.json`'s `mods[]`: `archive` and `sha256` from steps 2-3,
   `signature` - the same value stored in `manifest.json` inside the archive (its `signature`
   field). Add `icon`/`shortestDescription` (and `descriptions` if you have translations) too -
   see "catalog.json structure" above.
5. If this is a new author, add them to `authors[]` (id + public key).
6. Commit and push the updated `catalog.json`.

### Icon template

[`icon-template-1000x1000-circle.svg`](icon-template-1000x1000-circle.svg) at the root of this repo
is a starting point for a mod icon. Borg.Box fits your SVG icon's own bounding box into a circle of
radius 450 inside a 1000x1000 viewBox (no forced fill - your original colors are kept), so the
template includes a dashed guide circle at that same radius: draw your icon so nothing important
falls outside it, or it will get cropped by the automatic fit. Replace the `your-artwork-here` group
with your own artwork and remove the guides before using it.

---

## Русский

Источник модов для STFC Borg.Box. Единственный обязательный файл — `catalog.json` в корне этого
репозитория.

### Ссылка на источник

Всегда используйте прямую (raw) ссылку, не страницу просмотра файла на GitHub:

```
https://raw.githubusercontent.com/Plurimus/stfc-mod-source/refs/heads/main/catalog.json
```

Эту ссылку нужно указать в двух местах:
1. В Borg.Box → настройки центрального узла → «Мод-источники» → «Добавить по ссылке».
2. В Borg.Box → модуль сборки → поле «Ссылка на источник мода» — при сборке **каждого** своего
   мода (это привязка к тому, где проверять доверие, а не к тому, откуда скачан конкретный файл).

Borg.Box также требует хотя бы ОДИН локальный источник-папку (настройки центрального узла →
«Мод-источники» → «Локальная папка») — приложение скачивает копию мода в эту папку перед
установкой и проверяет её именно оттуда (см. «Трёхуровневая валидация» ниже). Одной только ссылки
на каталог для установки недостаточно.

### Структура catalog.json

- `authors[]` — доверенные авторы. `publicKeys[]` — список публичных ключей автора (см. ниже про
  несколько подписей), каждый ключ получаете кнопкой «Сгенерировать новый ключ» в модуле сборки
  Borg.Box. В `catalog.json` попадает только `pem` **публичного** ключа, приватный — никогда.
- `mods[]` — по одной записи на **версию** мода (Borg.Box показывает по кружку на версию,
  выстроенных в цепочку-ветку на один mod id). Поля:
  - `archive` — абсолютная или относительная (от самого catalog.json) ссылка на **один** файл
    `*.mod` (манифест уже лежит внутри него, отдельного файла манифеста Borg.Box не создаёт).
    Хранить сам `.mod` в этом репозитории не обязательно — удобно держать в assets релиза
    репозитория мода.
  - `sha256`/`signature` — см. «Как добавить свой мод» ниже.
  - `icon` — SVG-иконка в base64 (та же конвенция вписывания по bbox в круг, что и в поле иконки
    модуля сборки, см. «Шаблон иконки» ниже). Рекомендуется: тогда кружок НЕустановленного мода
    в дереве показывает реальную иконку, не скачивая архив целиком. Без неё — общая заглушка.
  - `shortestDescription` — рекомендуется: показывается в тултипе/панели узла. Без неё используется
    `summary`.
  - `descriptions` — необязательные переводы по языкам, например `{"ru": {"summary": "...",
    "shortestDescription": "..."}}`. Borg.Box берёт запись текущего языка интерфейса, если она
    есть, иначе — сами языконезависимые `summary`/`shortestDescription` мода. Добавляйте только те
    языки, для которых у вас есть настоящий перевод — это необязательное поле.

### Структура архива мода (`.mod`)

Файл `.mod` — обычный zip-архив (про добавленные после него байты см. «Трейлер архива» ниже) с
`manifest.json` в корне и всеми файлами, на которые этот манифест ссылается, каждый ровно по тому
пути, что указан в манифесте:

```
manifest.json
Your.Plugin.dll           # указан в installFiles[].source
README.md                 # указан в instructionsFile (необязательно)
changelog.md              # указан в changelogFile (необязательно)
Screenshot1.png           # указан в screenshots[] (необязательно)
```

Поля `manifest.json`:

- `id`, `name`, `version`, `authorId` — идентичность; `id` — постоянный ключ, по которому Borg.Box
  отслеживает установку и сравнивает версии, держите его неизменным между версиями одного мода.
- `type` — `0` для плагина BepInEx (ставится в `BepInEx/plugins`), `1` для патча сообщества
  (ставится в корень клиента).
- `summary`, `shortestDescription` — см. «Структура catalog.json» выше (те же поля, синхронизированы
  между манифестом и записью каталога).
- `instructionsFile` — путь (внутри этого архива) к Markdown-файлу с полным описанием/инструкцией
  — показывается во вкладке README в Borg.Box после успешной загрузки. Необязательно.
- `changelogFile` — путь (внутри этого архива) к ОТДЕЛЬНОМУ Markdown-файлу со списком изменений
  этой версии — показывается в собственной вкладке «Changelog» рядом с README (независимо от
  `instructionsFile` — мод не обязан иметь оба поля сразу). Необязательно.
- `icon` / `iconSvgBase64` — предпочтительно `iconSvgBase64` (тот же base64 SVG, что и поле `icon`
  в `catalog.json`, встроен прямо в манифест, а не как путь в архиве) — `icon` — устаревший вариант
  по пути в архиве.
- `screenshots[]` — пути (внутри этого архива) к скриншотам. Необязательно.
- `installFiles[]` — `{"source": "<путь в этом архиве>", "target": "<путь в папке plugins/корне
  клиента игры>"}` для каждого реально устанавливаемого модом файла.
- `minGameVersion` — необязательная минимальная версия клиента игры.
- `fileHashes[]` — SHA-256 каждого файла в архиве, кроме самого `manifest.json` (покрывает
  `installFiles`, `instructionsFile`, `changelogFile`, `screenshots`, `icon`), вычисляется
  инструментом подписи при упаковке — см. «Трёхуровневая валидация» ниже.
- `signature` — см. «Как добавить свой мод» ниже.

### Трёхуровневая валидация

Borg.Box проверяет мод в трёх точках, чтобы битый или подменённый файл обнаружился как можно
раньше:

1. **До скачивания** (для источников по ссылке): прямо из `catalog.json` — id, имя, версия,
   описание, иконка, `sha256`, `signature`, публичные ключи автора.
2. **После скачивания, до распаковки**: как только копия сохранена в локальном источнике-папке,
   Borg.Box читает её оттуда и сверяет имя/версию/автора/sourceRef с собственным **трейлером**
   архива (см. ниже) — дешёвая проверка без распаковки.
3. **После распаковки**: подпись уровня манифеста внутри `manifest.json` проверяется по публичным
   ключам автора (см. смену ключа ниже), и каждая запись `fileHashes` сверяется с реально
   распакованными байтами.

### Трейлер архива (MODTRL03)

Каждый `.mod`, собранный Borg.Box, несёт небольшой бинарный трейлер после zip-содержимого — в
первую очередь чтобы пункт 2 выше работал без распаковки: magic-байты (`MODTRL03`), `authorId`,
`sourceRef`, `name`, `version`, `shortestDescription`, `icon` (SVG в base64) и подпись уровня
архива — всё это зеркалирует `manifest.json`/`catalog.json`, а не отдельный источник истины.
Трейлер присутствует ВСЕГДА, даже в неподписанной сборке (пустая подпись), поэтому любой
инструмент может по последним байтам понять «это мод Borg.Box», не трогая сам zip.

### Несколько подписей на одного автора (смена ключа)

Один автор может иметь **несколько** ключей одновременно в `publicKeys[]` — это нужно на случай,
если приватный ключ потерян (диск сломался, ключ не сохранён и т.п.), но доступ к самому
репозиторию-источнику сохранился. В этом случае:

1. Сгенерируйте НОВЫЙ ключ в Borg.Box («Сгенерировать новый ключ»).
2. Добавьте его как ЕЩЁ ОДИН элемент в `publicKeys[]` того же автора — **не удаляя** старые ключи.
   Пометьте новый как `"status": "active"`, у старого(-ых) смените на `"status": "revoked"`
   (`revoked` — чисто информационная пометка «этим ключом больше не подписываю новое», она НЕ
   мешает проверке старых, уже выпущенных модов).
3. Все НОВЫЕ моды подписывайте новым ключом.
4. Старые моды, подписанные прежним ключом, останутся проверяемыми: при проверке подписи нужно
   пробовать **все** ключи автора по очереди (независимо от `status`), пока один не подойдёт — так
   старые плагины не потеряют доверенный статус только из-за смены ключа.

Ключи из `publicKeys[]` никогда не удаляются насовсем (иначе перестанут проверяться уже
опубликованные версии модов) — только помечаются `revoked`, если хотите визуально показать, что
ключ сменился.

### Как добавить свой мод

1. Соберите и подпишите мод в Borg.Box (кнопка «Собрать мод», с загруженным ключом подписи и
   заполненной ссылкой на источник — см. выше). На выходе один файл — `<id>-<version>.mod`.
2. Выложите этот файл как asset релиза в репозитории мода, например:
   ```
   https://github.com/<author>/<mod-repo>/releases/download/v1.0.0/<name>-1.0.0.mod
   ```
3. Посчитайте SHA-256 файла `*.mod` (например, `certutil -hashfile file.mod SHA256` в Windows).
4. Добавьте/обновите запись в `mods[]` этого `catalog.json`: `archive` и `sha256` — из шагов 2-3,
   `signature` — то же значение, что лежит в `manifest.json` внутри архива (поле `signature`).
   Добавьте также `icon`/`shortestDescription` (и `descriptions`, если есть переводы) — см.
   «Структура catalog.json» выше.
5. Если это новый автор — добавьте его в `authors[]` (id + публичный ключ).
6. Закоммитьте и запушьте изменения в `catalog.json`.

### Шаблон иконки

Файл [`icon-template-1000x1000-circle.svg`](icon-template-1000x1000-circle.svg) в корне репозитория —
заготовка для иконки мода. Borg.Box сам вписывает вашу SVG-иконку по её собственному bbox в круг
радиуса 450 внутри viewBox 1000×1000 (без принудительной заливки — исходные цвета сохраняются),
поэтому в шаблоне нанесена пунктирная направляющая окружность того же радиуса: рисуйте иконку так,
чтобы важные детали не попадали за эту границу, иначе они будут обрезаны при автоматической подгонке.
Замените группу `your-artwork-here` своим рисунком и уберите направляющие перед использованием.
