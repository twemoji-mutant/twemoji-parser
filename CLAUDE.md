# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

twemoji-parser (`@twemoji/parser`) is a library that identifies emoji entities within a string and returns their positions, text, and Twemoji CDN URLs. The project has two major parts: a **Scala regex generator** that produces an optimized emoji-matching regex, and a **JavaScript parser** that uses that regex to find emoji in text.

## Commands

### JavaScript (from repo root)
- **Install:** `yarn install`
- **Test:** `yarn test` (Jest, jsdom environment)
- **Test single:** `yarn test -- --testPathPattern=<pattern>`
- **Test watch:** `yarn test:watch`
- **Lint:** `yarn lint` (ESLint with Prettier/Flow/Jest plugins)
- **Type check:** `yarn flow`
- **Build:** `yarn build` (Babel transpile `src/` to `dist/`, ignores test files)
- **Full CI:** `yarn ci` (lint + flow + test)

### Scala regex generator (from `src/scala/`)
- **Compile:** `./pants compile ::`
- **Test:** `./pants test ::`
- **Regenerate regex:** `./src/scala/scripts/generate.sh` (from repo root)
  - Runs the Scala generator then `yarn lint` to format the output

Scala requires Java 8, Scala 2.12, and Python 3.8 in PATH. First build takes several minutes; subsequent builds use cache.

## Architecture

### Regex generation pipeline (the core of this project)

The emoji regex in `src/lib/regex.js` is **generated code — do not edit manually**. It is produced by the Scala program in `src/scala/` via this pipeline:

1. **`src/scala/config/src/main/resources/config/emoji.yml`** — Master emoji data (~6800 lines). Each emoji has `unicode` (hex codepoints), `description`, `keywords`, and a `type` field that determines how it's processed. To add/update emoji support, edit this file.

2. **Emoji types** (defined in `YamlParser.scala` as `EmojiType` enum): `Normal`, `Keycap`, `Flag`, `Regional`, `Variant`, `Directional`, `Diversity`, `DirectionalDiversity`, `VariantDiversity`, `TextDefault`, `MultiDiversity`. The type controls how skin tones, variation selectors, and directional modifiers are applied.

3. **`Item.scala`** — Represents a single emoji. Computes all variant sequences (skin tones, directional modifiers) and `maxCodePointSequenceLength` based on emoji type.

4. **`EmojiInfoGeneratedView.scala`** (~430 lines) — Core regex construction logic:
   - Separates emoji into categories: multiDiversity, ZWJ (with/without gender), keycap, variant, textDefault, diversity, directional, normal
   - Optimizes regex by grouping codepoint sequences with shared prefixes (e.g., `abc|abd|abe` → `ab[cde]`) and collapsing contiguous ranges into character classes
   - Converts codepoints to UTF-16 surrogate pairs for JavaScript (`isUCS2: true`)
   - Line-wraps patterns at 85 characters

5. **`src/scala/generator/src/main/resources/codegen/regex.js.mustache`** — Mustache template that assembles all regex fragments into the final JavaScript regex, ordering patterns by priority (multi-diversity first, then ZWJ sequences, then keycaps, variants, diversity, flags/normal, and finally bare VS16).

6. **`Main.scala`** — Entry point. Loads emoji.yml, creates `EmojiInfoGeneratedView`, renders through Mustache, writes to `src/lib/regex.js`.

### JavaScript parser

- **`src/index.js`** — Exports `parse(text, options?)` and `toCodePoints()`. Uses Flow types. Zero runtime dependencies.
  - `parse()` runs the generated regex against input text, converts matches to codepoint strings (removing VS16 except in ZWJ sequences), and builds CDN URLs
  - Default CDN: `https://cdn.jsdelivr.net/gh/jdecked/twemoji@latest/assets/svg/{codepoints}.svg`
  - Options: `assetType` ('svg'|'png'), `buildUrl` (custom URL builder)
- **`src/__tests__/index.test.js`** — ~1066 lines. Extensive tests organized by Unicode Emoji version (2.3 through 17.0), covering skin tones, ZWJ sequences, directional variants, flags, and multi-diversity combinations.
- **`index.d.ts`** — TypeScript declarations (manually maintained, separate from Flow types).
- **`dist/`** — Build output (Babel-compiled CommonJS). Published to npm along with `index.d.ts`.

## Code Style

- Flow type annotations (`// @flow` at top of source files)
- Prettier: single quotes, 120 char print width
- Tests must use `test()`, not `it()` (enforced by ESLint `jest/consistent-test-it`)
- Snapshots are disallowed (`jest/no-large-snapshots` maxSize: 0)
- `prefer-const`, `no-var`, `prefer-template` enforced
- Imports must be sorted (`sort-imports` rule)
- PR format: Problem / Solution / Result sections (`.github/PULL_REQUEST_TEMPLATE.md`)
- Commit messages: scoped subjects, 72 char max (see `CONTRIBUTING.md`)
