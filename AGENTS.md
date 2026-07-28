# AGENTS.md

Notes for agents working in this repo. The project is a single unpacked
browser extension plus prebuilt archives; there is no build step, bundler,
or test suite.

## Layout

- `chrome-extension/` — the extension source. This directory *is* the package;
  its contents go at the root of the archive.
- `dist/` — prebuilt archives, one `.zip` (Chrome) and one `.xpi` (Firefox)
  per released version, plus `sona-emotion-sprites-latest.zip`.
- `scripts/aggregate_model_emotions.py` — unrelated to packaging; queries
  OpenRouter models for emotion-state names.

## Repackaging

Everything below assumes you are releasing version `X.Y.Z`. Run the commands
from the repository root.

### 1. Bump the manifest

`chrome-extension/manifest.json` is the single source of truth for the version.
Update `"version"` there first — the archive filenames must match it.

### 2. Build the archive

The archive is zipped from **inside** `chrome-extension/`, so `manifest.json`
lands at the archive root. Zipping the directory itself produces a
`chrome-extension/` prefix and the browser will reject it.

```sh
VERSION=$(python3 -c 'import json;print(json.load(open("chrome-extension/manifest.json"))["version"])')
rm -f "dist/sona-emotion-sprites-$VERSION.zip"
(cd chrome-extension && zip -r "../dist/sona-emotion-sprites-$VERSION.zip" . -x '.DS_Store' '*/.DS_Store')
```

### 3. Copy to the `.xpi` and `latest.zip`

An `.xpi` is just a zip with a different extension; the released `.xpi` and
`.zip` for a version are byte-identical. `latest.zip` is a copy of the newest
versioned zip, not a separate build.

```sh
cp "dist/sona-emotion-sprites-$VERSION.zip" "dist/sona-emotion-sprites-$VERSION.xpi"
cp "dist/sona-emotion-sprites-$VERSION.zip" dist/sona-emotion-sprites-latest.zip
```

### 4. Whitelist the new zip in `.gitignore`

`dist/*` is ignored and each tracked archive is un-ignored by name. A new
release will be invisible to git until you add a line:

```
!dist/sona-emotion-sprites-X.Y.Z.zip
```

Only the `.zip` files are tracked. The `.xpi` files are deliberately left
untracked — do not `git add -f` them.

### 5. Update the README pointer

`README.md` names the current build next to the `latest.zip` link:

> Latest ZIP: `dist/sona-emotion-sprites-latest.zip`, currently the same build
> as `dist/sona-emotion-sprites-X.Y.Z.zip`.

Bump that version so the link resolves. A link to an archive that was never
whitelisted in step 4 is a 404 on GitHub.

### 6. Verify before committing

```sh
unzip -p "dist/sona-emotion-sprites-$VERSION.zip" manifest.json | head -5   # version matches
unzip -l "dist/sona-emotion-sprites-$VERSION.zip" | head                     # no chrome-extension/ prefix
cmp "dist/sona-emotion-sprites-$VERSION.zip" dist/sona-emotion-sprites-latest.zip
git status --porcelain                                                       # new zip shows as added
```

Then commit the manifest, `.gitignore`, `README.md`, the new zip, and
`latest.zip` together.

## Rules

- **Never rewrite a released archive.** If a source change lands after
  `dist/sona-emotion-sprites-X.Y.Z.zip` was committed, bump the version and
  build a new one. Rebuilding an existing versioned archive in place leaves a
  file whose manifest claims one version while its code is something else.
  If you find such a rebuild in the working tree, `git restore` it.
- Keep old archives. Superseded builds (including the Claude-only
  `claudesona-emotion-sprites-1.0.1.zip`) stay in `dist/` for reference.
- `chrome-extension/README.md` ships inside the archive and lists the
  supported tags. If you add or rename an emotion tag in `content.js`, update
  that file and the tag lists in the top-level `README.md`.
- Adding a new emotion means adding `chrome-extension/assets/<prefix>_<name>.png`
  as well; the sprite filename is derived from the tag name.

## Manual smoke test

There are no automated tests. To check a build: `chrome://extensions` →
Developer mode → **Load unpacked** → select `chrome-extension/` (the inner
folder, not the repo root), then open `https://claude.ai`,
`https://chatgpt.com`, or `https://gemini.google.com` and type a supported tag
such as `<claude_curious />` into a message. Tags are host-specific — `claude_*`
tags only render on `claude.ai`, and so on.
