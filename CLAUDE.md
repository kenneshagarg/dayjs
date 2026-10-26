# CLAUDE.md

Day.js: a ~3 KB immutable date library that copies Moment.js's API. Core stays tiny; everything optional is a plugin or a locale. Matching Moment's output is the correctness bar.

## Layout
- `src/index.js` — the `Dayjs` class and `dayjs()` factory. Instance state lives in `$`-prefixed fields (`$d` native Date, `$y $M $D $H $m $s $ms $W` cached parts, `$L` locale, `$u` utc flag, `$x` plugin extras).
- `src/constant.js`, `src/utils.js` — unit names (`C.M`, `C.D`…), regexes, helpers.
- `src/plugin/<name>/index.js` — one dir per plugin, `export default (option, DayjsClass, dayjsFactory) => {}`. Patch `DayjsClass.prototype`; reach helpers via `this.$utils()` or `dayjsFactory.Ls/.en/.p`.
- `src/locale/<code>.js` — one file per locale (143). Imports `dayjs` (mapped to `src` by `test/__mocks__/dayjs.js`).
- `types/` — hand-written `.d.ts`; `types/plugin/<name>.d.ts` per plugin. Copied to the root at build.
- `test/` — jest, mirrors `src/` (`test/plugin/<name>.test.js`, `test/locale/`). `test/issues/` holds regression tests named `issueNNNN.*.test.js`.
- `build/` — rollup scripts that bundle each locale and plugin separately.

## Commands
```sh
npm ci
TZ=UTC npm test                                        # full suite, as CI runs it
npx jest test/plugin/isToday.test.js --coverage=false  # one file
npm run lint                                           # eslint on src/ test/ build/
NODE_OPTIONS=--openssl-legacy-provider npm run build   # bundles + 2.99 KB size check
```
- `npm test` re-runs the timezone suites 8 times under different `TZ` values, then the whole suite with a **100% line-coverage threshold**. Untested new lines fail it.
- Always prefix `TZ=UTC`: one timezone pass and the final full run inherit the machine's zone, and on a US-timezone laptop 6 tests fail on a clean checkout (dayOfYear, localizedFormat, timezone).
- Without `--coverage=false`, a single-file run prints the coverage table for all of `src/`.
- Without the `NODE_OPTIONS` flag, `build` crashes in its `size` step on Node ≥17 (`ERR_OSSL_EVP_UNSUPPORTED`) after writing the bundles.
- The size step prints two numbers. Only `Package size` vs `Size limit` is enforced (2.79 KB on `dev`). The trailing bare `3.04 kB` is `gzip-size` output, informational, and also 3.04 on a clean checkout. Don't stash or diff to investigate it.

## Conventions
- New plugin = `src/plugin/<name>/index.js` + `test/plugin/<name>.test.js` + `types/plugin/<name>.d.ts`. `test/plugin/declarations.test.js` fails if the `.d.ts` is missing.
- New locale: first line must be `// <English name> [<code>]`; `name` must equal the lowercase filename. `test/locale/keys.test.js` enforces both; the build reads that comment.
- Tests import from `'../../src'` and register plugins with `dayjs.extend(...)`. Pin time with `MockDate.set(...)` in `beforeEach` and `MockDate.reset()` in `afterEach`. Assert against `moment` when moment has the same API, e.g. `expect(dayjs(x).format()).toBe(moment(x).format())`.
- Plugins import only from `../../constant` (or a sibling plugin's `utils`), never `../../index` or `../../utils`, because that would bundle core into every plugin file.
- Style: airbnb-base, **no semicolons**, no trailing commas, no import extensions. Commits: `fix(plugin): …`, `feat: …`, `chore: …` (semantic-release cuts versions from these).

## Guardrails
- `/plugin`, `/locale`, `/esm`, `/dayjs.min.js`, `/index.d.ts`, `/locale.json` at the root are **build output** (gitignored). Edit `src/` or `types/` instead.
- Grow `src/index.js` only when unavoidable; the 2.99 KB gzip limit is checked on build. Prefer a plugin.
- `docs/` is translated README stubs; real API docs live at day.js.org (separate repo). Don't edit them for API changes.
- Base PRs on `dev`, not `master`: CI runs on `dev`, and pushes to `master` trigger an npm release.
- Don't bump the pinned `moment`/`moment-timezone` versions; tests compare against their exact output.
