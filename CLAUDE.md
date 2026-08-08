# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is the **source** repository for `typo3scan`, a CLI tool that scans a code base for usage of deprecated/breaking TYPO3 APIs. It only *reports*; it never rewrites code (for automatic fixes, users are pointed at TYPO3 Rector).

Two things follow from that:

- The compiled artifact (`typo3scan.phar`) is built here and then copied into a **separate binary release repository** cloned at `../typo3scan`. That repo holds the distributed binary and the user-facing documentation.
- The actual matcher *rules* do **not** live here. They live in the scanner library (a fork of the scanner extracted from the TYPO3 v9 core, hosted at `https://github.com/mschwemer/scanner`). Fixing a false positive or adding a new matcher almost always means changing the scanner library, not this repository.

### The scanner package was renamed

The fork was relicensed and renamed: the package is **`m12r/typo3-cms-scanner`** and the namespace is **`M12R\Scanner\`** (previously `typo3/cms-scanner` / `TYPO3\CMS\Scanner\`), with a PHP 8.3 floor. This repository tracks its `main` branch as `dev-main` through a normal `vcs` repository entry, so `composer update m12r/typo3-cms-scanner` pulls scanner changes as usual.

The class layout and `config/Matcher/` structure are unchanged from the TYPO3 original, so pre-rename knowledge of the scanner still applies — only the vendor namespace differs. Older material (including this tool's git history and the release repo's docs) will still refer to `TYPO3\CMS\Scanner`.

## Commands

```bash
composer install                      # installs deps; note bin-dir is ./bin
./bin/typo3scan scan ~/path/to/code   # run from source
./bin/typo3scan scan --target 12 --format markdown --only breaking ~/path/to/code

composer update m12r/typo3-cms-scanner  # pull in scanner library changes before building
composer bin box install              # REQUIRED before the first build; see below
composer build                        # box compile -> typo3scan.phar, then copies it to ./typo3scan
composer prep                         # copies both phar files to ../typo3scan (the release repo)
```

**Box is not installed by `composer install`.** `bin/box` is only a Composer proxy pointing at `vendor-bin/box/vendor/humbug/box/bin/box`, which `composer install` never populates — so `composer build` fails with an `include(): Failed to open stream` warning until you run `composer bin box install` once per checkout. `vendor-bin/box/composer.json` (a single `humbug/box` requirement) *is* tracked; the rest of `/vendor-bin` is gitignored via a narrow negation rule in `.gitignore`, so do not "tidy" that rule into a plain `/vendor-bin` — that would hide the file again and the build would stop being reproducible.

The install reports `Skipped installation of bin ... name conflicts with an existing file` for `box`, `php-scoper`, `php-parse` and others — expected and harmless, since the proxies in `bin/` are already correct.

Box needs `phar.readonly=0`; it restarts itself with that setting, so no php.ini change is needed.

**Exit codes matter and are load-bearing for CI use:** `scan` returns `1` when *any* match is found and `0` when the code is clean. Do not "fix" this to always return 0.

There is **no test suite in this repository.** Tests belong to the scanner library and are run from that checkout:

```bash
cd ~/path/to/scanner-library && composer install && ./.Build/bin/phpunit tests/
```

### Working on the scanner library alongside this tool

As a **temporary local override only**, add a `path` repository as the first entry in `composer.json` `repositories` pointing at the local scanner clone, then `composer update`. Composer symlinks the library so edits are reflected immediately when running `./bin/typo3scan`.

```json
{ "type": "path", "url": "~/Projects/TYPO3/scanner/" }
```

Revert this before committing — the committed state resolves the scanner from the `vcs` repository, never from a local path.

## Architecture

Only three PHP classes exist; the bulk of the repository is vendored TYPO3 changelog data.

**`bin/typo3scan`** — entry point. Probes three autoload locations so the same file works as a Composer dependency, as a standalone checkout, and inside the phar. Suppresses `E_NOTICE|E_WARNING|E_DEPRECATED` (scanning code written for old TYPO3 versions otherwise floods the output). **The application version string is hardcoded here** (`new Application('TYPO3Scan', '1.7.5')`) — bump it here when releasing.

**`Service/ScannerService.php`** — translates `--target <version>` into a `MatcherBundleCollection`. Each supported major version gets its own `case` listing exactly which `Matcher\*` classes apply, because older TYPO3 versions have fewer matcher types. It resolves the rule files from `vendor/m12r/typo3-cms-scanner/config/Matcher/v<version>/` via `setMatcherBundlePath()`, which probes two relative paths — one for a phar/standalone layout, one for when this package is itself installed under someone else's `vendor/`. These are **plain strings, not imports**, so they are invisible to any namespace-wide search-and-replace; if the scanner package is ever renamed again, updating the `use` statements alone will leave the tool reporting "0 matches" with no error. `case '7'` is also the `default`, so an unknown `--target` silently scans against v7 rules.

**`Command/ScanCommand.php`** — everything else. The flow is: scan → three filter passes (`filterByType` for `--only`, `filterByIndicators` for `--strong/weak`, `filterByIssuenumbers` for `--issue`/`--ignore`) → count/percentage statistics → Twig render. Each filter rebuilds a fresh `DirectoryMatches`/`FileMatches` tree rather than mutating in place.

The class also registers all Twig filters used by the templates. Two of them do the non-obvious work of turning a matcher's `restFiles` reference into human-readable output:

- `getChangeTitle` → `findRstFile()` builds a one-time basename→path index over `src/Resources/Private/Changelog/**/*.rst` and then parses the title out of the file. It matches on **basename only**, because matchers are inconsistent about including the version directory in the path they report.
- `getOnlineDocument` → matches the basename against `src/Resources/Private/links.txt` (~3700 lines) to build a `docs.typo3.org` URL.

Both hardcode special cases (with `crc32()` cache keys) for the v7 `ClassNamePatternMatcher` rules, which reference pseudo-files `Deprecation-legacy-files.md` / `Deprecation-non-namespaced.md` that have no real rst counterpart.

**Gotcha:** if the scanner library references an rst file whose basename is absent from `src/Resources/Private/Changelog/`, `findRstFile()` hits an undefined array key and the report degrades silently (the warning is suppressed by `bin/typo3scan`). Keep the vendored changelogs in sync with the scanner library's rule files.

**`src/Resources/Private/Templates/*.twig`** — output formats. The template is selected by `ucfirst($format) . '.twig'`, so `--format markdown` loads `Markdown.twig`. Adding a format means adding a template file; no code change is needed. `--templatePath` prepends a user directory to the Twig loader, so users can override any template by filename.

## Adding support for a new TYPO3 major version

1. Copy that version's changelog `.rst` files from the TYPO3 core into `src/Resources/Private/Changelog/<version>/`.
2. Append the corresponding documentation paths to `src/Resources/Private/links.txt`.
3. Add a `case '<version>':` to `ScannerService::__construct()` with the matcher list for that version (new matcher classes must exist in the scanner library first).
4. Update the `--target` option default in `ScanCommand::configure()`.

Commits `61c5dd6` and `1b13976` are the reference for this sequence (v14 support).

## Conventions

- Commit messages follow the existing prefix style: `[TASK]`, `[FEATURE]`, `[BUGFIX]`, `[CLEANUP]`.
- The package is `m12r/typo3scan` and the namespace is `M12R\TYPO3Scan\` (both formerly `michielroos` / `MichielRoos`). The PSR-4 root still maps to `src/TYPO3Scan` — only the vendor segment changed, so do not rename that directory. Michiel Roos' author entry in `composer.json`, the `.github/FUNDING.yml` link, the per-file MIT copyright headers, and the credit in the CLI banner are **attribution and stay as they are**; they are not leftovers of the rename.
- The existing source predates modern TYPO3 CGL — no `declare(strict_types=1)`, docblock-based typing, `protected` helpers on the command class. Match the surrounding file when editing it; apply stricter CGL to genuinely new files rather than churning the existing three classes.
- `bin/` is gitignored except for `bin/typo3scan` (Composer's `bin-dir` is `bin`, so `composer install` drops `box`, `php-scoper` etc. there). `typo3scan.phar` is force-tracked despite the ignore rules.
