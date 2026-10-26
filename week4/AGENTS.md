# AGENTS.md

## What this is
Day.js is a ~2KB immutable date-time library with a Moment.js-compatible API.
Core features stay minimal; everything else ships as opt-in locales and plugins
(`dayjs.extend(plugin)`, `dayjs.locale('xx')`).

## Architecture
- `src/index.js` – core: the `Dayjs` class, `dayjs()` factory, global locale
  registry, `extend`, `locale`. `src/utils.js` and `src/constant.js` are shared helpers.
- `src/plugin/<name>/index.js` – one directory per plugin. A plugin is
  `export default (option, DayjsClass, dayjsFactory) => { ... }` that patches
  `DayjsClass.prototype`.
- `src/locale/<code>.js` – one file per locale; first line is a
  `// Language Name [code]` comment (parsed by `build/index.js` into `locale.json`).
- `types/` – hand-written TypeScript declarations (`index.d.ts`, `plugin/*.d.ts`,
  `locale/`). Update these when changing a public API.
- `build/` – Rollup (UMD, minified) build scripts; `build/esm.js` post-processes the ESM output.
- `docs/` – translated READMEs per language.

## Commands
```sh
npm ci               # reproducible install from package-lock.json
npm run lint         # eslint src/* test/* build/* (also runs as a pre-commit hook)
npm test             # timezone suites under several TZs, then full jest with 100% line coverage
npx jest test/plugin/utc.test.js --runInBand --coverage=false  # focused test
npm run build        # Rollup -> dayjs.min.js, locale/, plugin/, copies types/, then size-limit
npm run babel        # ESM build -> esm/

Day.js is a library, not a standalone application. There is no `npm start` script. Validate changes with the test suite and build commands.
```
CI (`.github/workflows/lint-test.yml`) runs `npm run lint` then `npm test` on PRs to `dev`.

## Tests
- Jest, `test/**/*.test.js`. Core tests in `test/*.test.js`, plugins in
  `test/plugin/<name>.test.js`, locales in `test/locale/<code>.test.js`,
  regressions in `test/issues/`.
- Tests usually compare against `moment` / `moment-timezone` and freeze time with
  `MockDate` in `beforeEach`/`afterEach`.
- `test/__mocks__/dayjs.js` maps the bare `dayjs` import (used by locale files)
  to `src/`.
- Timezone-sensitive: `test/timezone.test.js` runs under `TZ=Pacific/Auckland`,
  `Europe/London`, `America/Whitehorse` and host TZ; `test/plugin/timezone.test.js`
  runs under `Europe/Paris`, `Europe/London`, `America/New_York`, `Pacific/Auckland`
  (via `cross-env`, see `package.json`). Reproduce a TZ bug with e.g.
  `npx cross-env TZ=Pacific/Auckland npm run test-tz`.
- Coverage threshold is 100% lines for `src/**`; new code needs tests.
- `test/browser.spec.js` is for Sauce Labs/Karma and says "Please do NOT modify".

## Conventions
- ES6, ESLint `airbnb-base` with: no semicolons, no trailing commas, no import
  file extensions. 2-space indent, LF, final newline (`.editorconfig`).
- Semantic (Conventional) commit messages, e.g. `fix(plugin): ...`, `feat: ...`,
  `docs: ...`, `chore: ...` – semantic-release derives versions and `CHANGELOG.md` from them.
- PRs target `dev`; releases are cut from `master`.
- Keep changes narrowly scoped. Explain the proposed files and approach before editing.
- Run the narrowest relevant test first, then lint and broader tests as appropriate.

## Do not edit
- Build outputs (gitignored): `dayjs.min.js`, `/locale`, `/plugin`, `/esm`,
  root `/index.d.ts`, `locale.json`. Edit `src/` and `types/` instead.
- `CHANGELOG.md` – written by semantic-release.
- `coverage/`, `node_modules/`, `package-lock.json` (unless changing deps),
  `patches/` (patch-package fix for `@semantic-release/github`).

## Watch out
Don't "fix" `import dayjs from 'dayjs'` in `src/locale/*` into a relative
path like `'../index'`. Rollup treats `dayjs` as external so each locale
bundle stays separate from core; a relative import would inline the whole core
into every bundle. Also keep the core small – `npm run build` enforces a
2.99 KB `size-limit` on `dayjs.min.js`, so new features usually belong in a plugin.
