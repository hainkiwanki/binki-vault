# Binki Project Review

_Date: 2026-10-24 · Scope:_ `apps/api`_,_ `apps/server`_,_ `apps/plex`_,_ `apps/ingest-extension`_,_ `apps/shared`_, Dockerfiles, compose files, CI, tooling · Mode: read-only review, no code changed_

Four specialist reviewers looked at the monorepo independently: a **TypeScript Engineer**, a **Senior Vue Developer**, a **Security Engineer** and a **Backend API Engineer**. Each one checked performance, over-engineering, security, design-pattern use and clean code from their own angle. This document has a combined summary first, followed by each reviewer's full report.

> **Deployment context:** Binki runs only on a home server and is reachable only through a WireGuard tunnel. Security severities are rated for that setup. Where a rating would change if the app were ever exposed publicly (for example through a Cloudflare Tunnel), the finding says so.

Github edit

---

## Table of Contents

1. [Executive summary](#executive-summary)
2. [Cross-cutting themes](#cross-cutting-themes)
3. [Consolidated priority list](#consolidated-priority-list)
4. [TypeScript Engineer](#typescript-engineer)
5. [Senior Vue Developer](#senior-vue-developer)
6. [Security Engineer](#security-engineer)
7. [Backend API Engineer](#backend-api-engineer)

---

## Executive summary

At compile time the codebase is healthy:

- `strict` is on everywhere and there is effectively one `any`.
- All packages type-check and `eslint --max-warnings=0` passes.
- SQL is fully parameterized and API keys never reach the browser.
- The ingest fetcher has a strict SSRF allowlist.
- The optional-dependency boot with feature gating is well thought out.

The problems are at the boundaries and in runtime behaviour:

- **Contracts drift.** The media-portal API contract (Zod schemas in `apps/api`) is copied by hand into `apps/plex`, and the copies have already drifted. Upstream and HTTP JSON is cast with `as T` instead of being validated. The Vue apps are never type-checked in CI.
- **Requests fan out.** Each `MediaCard` makes its own `/media-info` request. The catalog sync sends one Jellyfin request per library item at the same time. Genre posters and request enrichment make sequential or unbounded Jellyseerr calls. Nothing is cached.
- **Lifecycle and idempotency are weak.** There are no SIGTERM handlers, so graceful shutdown never runs in Docker. Ingest retries re-upload media under random S3 keys and insert duplicate rows. Long yt-dlp jobs can outlast RabbitMQ's consumer timeout.
- **Frontend state is fragile.** Settings edits write into the shared `DEFAULT_SERVER_SETTINGS` constant. Caches are never invalidated. Typing quickly in search shows a "Search failed" error.
- **The bundle is too big.** Vuetify is fully registered, which cancels tree-shaking. All routes load eagerly, so video.js ships on every page.
- **Security (even inside WireGuard):** the biggest real risk is yt-dlp following Reddit embeds through its generic extractor into the LAN and compose network. After that come DNS rebinding with an over-broad CORS rule, TLS verification being disabled for Yarn, and containers running as root.

## Cross-cutting themes

Several reviewers reported the same issues independently. These are the most reliable findings:

| Theme                                                                                                      | Reported by       | Key locations                                                                        |
| ---------------------------------------------------------------------------------------------------------- | ----------------- | ------------------------------------------------------------------------------------ |
| Error handler sends 5xx internals to clients (both ternary branches identical)                             | TS, Backend       | `apps/api/src/shared/errors/errorHandler.mts:5-23`                                   |
| `DEFAULT_SERVER_SETTINGS` mutated through `ref()`                                                          | TS, Vue           | `apps/plex/src/store/settingsServer.store.mts:9,53`                                  |
| `auth.userId` uses `!`, so the `=== undefined` guards are dead                                             | TS, Vue           | `apps/plex/src/store/auth.store.mts:12-13`                                           |
| Vuetify `import * as components` + eager routes + duplicate MDI font                                       | TS, Vue           | `apps/plex/src/plugins/vuetify.mts:4-10`, `plugins/router.mts`                       |
| Unbounded `hydrateAllMedia` fan-out in catalog sync                                                        | TS, Backend       | `apps/api/src/media-portal/services/jellyfin.service.mts:70-76`                      |
| N+1 inserts (catalog, tags, seasons)                                                                       | TS, Backend       | `binkiPlex.repo.mts:36-49`, `tags.repo.mts:3-27`                                     |
| Sequential genre-poster requests, no caching                                                               | TS, Backend, Vue  | `binkiPlex.service.mts:124-156`                                                      |
| Scattered, unvalidated `process.env` reads with `!`                                                        | TS, Backend       | 59 reads across 19 files; centralize in a Zod `config.mts`                           |
| Query params typed as strings and converted by hand                                                        | TS, Backend       | `media-portal/schemas/index.schema.mts:98,104,120,146`                               |
| Extension background uses unloaded or stale options                                                        | TS, Vue           | `ingest-extension/src/background/background.mts:5-10`, `optionsController.mts:13-16` |
| Webhook `?token=` ends up in logs; `X-Forwarded-For` trusted                                               | Security, Backend | `portalHooks.mts:123-126`, `requestLog.mts:16-26,61`                                 |
| Containers run as root, unpinned images and yt-dlp, no `.dockerignore`                                     | Security, Backend | `apps/*/Dockerfile`, compose files                                                   |
| Unbounded `tags` in the ingest contract                                                                    | Security, Backend | `apps/shared/src/ingest/ingest.contract.mts:27`                                      |
| Dead code (TMDB client, `QuickActionButtons.vue`, `Collection.vue`, legacy routes, duplicate SSE endpoint) | TS, Vue, Backend  | see each report                                                                      |
| `JOB_METRICS.durationMs` grows forever; sync `writeFileSync` stats every 5 s                               | TS, Backend       | `jobMetrics.mts:3-13`, `requestCount.mts`                                            |

## Consolidated priority list

1. **Network and supply-chain hardening.** Add a Host allowlist hook, pin the CORS rule to your extension ID, remove `enableStrictSsl: false`, add `USER node` and other container hardening, pin images and actions, and set a Valkey `--requirepass`. _(Security)_
2. **Fix extension reliability.** Await `loadOptions()` in `runIngestFlow`, clear the cache on `storage.onChanged`, add the `notifications` permission and icon, and move `decrementBadge()` into `finally`. _(Vue, TS)_
3. **Fix playback latency.** Decide direct play vs. HLS once and make `/stream` a pure range proxy. Return `202` from request endpoints and deliver the outcome over SSE. _(Backend)_

---

## TypeScript Engineer

**Summary.** At compile time the monorepo is in good shape. `strict` is on everywhere, there is a single `any` in the whole codebase, and every package type-checks cleanly with `tsc`/`vue-tsc --noEmit`. `eslint . --max-warnings=0` also passes. The weak points are at the boundaries:

- The Vue apps are never type-checked in CI.
- Almost every upstream or HTTP JSON payload is cast with `as T` instead of being validated, even though Zod is already installed.
- The media-portal contract is written as Zod schemas in `apps/api` and then re-typed by hand in `apps/plex`, and the copies have drifted. `apps/shared` is only used for the ingest contract and the Jellyfin playback types.
- The backend is less strict than the frontends (no `noUncheckedIndexedAccess`), and lint uses only the non-type-aware `typescript-eslint` preset.

### Findings

#### Tooling

**1. High – Vue apps are never type-checked.**

- Where: `apps/plex/package.json:12` (`"build": "vite build"`), `apps/ingest-extension/package.json:11`, `.github/workflows/build-lint.yml:49-53`.
- Problem: `vue-tsc` is installed but no script runs it. Vite strips types without checking them, and CI only runs `yarn lint` and `yarn build`, so type errors in plex and the extension ship silently.
- Fix: add a `"typecheck": "vue-tsc -b"` script per app (`tsc --noEmit` for api, server and shared), a `typecheck` task in `turbo.json` with `"dependsOn": ["^build"]`, and run `yarn turbo typecheck` in CI.

**2. High – Backend is less strict than the frontends.**

- Where: `tsconfig.base.json:2-11`. `@vue/tsconfig` turns on `noUncheckedIndexedAccess` for both Vue apps, but api, server and shared get neither that nor `noUnusedLocals/Parameters`, `noImplicitOverride`, `verbatimModuleSyntax` or `isolatedModules`.
- Problem: with the flag on, real bugs appear: `resource.repo.mts:46,52` (returns `rows[0]` as if it always exists), `tags.repo.mts:16` (`tagResult.rows[0].id`), `httpDownload.mts:39` and `binkiPlex.service.mts:313`. That is only 5 errors, so it is cheap to enable.
- Fix: add to the base config:

**3. High – ESLint is not type-aware.**

- Where: `eslint.config.mjs:26-27` uses `...ts.configs.recommended`.
- Problem: there is no `no-floating-promises`, `no-misused-promises`, `no-unsafe-*`, `switch-exhaustiveness-check` or `only-throw-error`. The Node block (`:31`) also skips `apps/server/src`.
- Fix: use `ts.configs.recommendedTypeChecked` with `parserOptions.projectService: true`. Add `apps/server/src/**` to the Node block. Turn on `no-non-null-assertion: 'warn'` and `consistent-type-assertions` with `objectLiteralTypeAssertions: 'never'`.

**4. Low – Dead or inconsistent compiler options.**

- `importHelpers: true` without `tslib` installed.
- `experimentalDecorators` with no decorators in the code.
- `esModuleInterop`/`resolveJsonModule` set twice in the extension.
- `allowImportingTsExtensions` differs between plex and the extension.
- `apps/shared` has no `types: []`.
- Fix: put one `tsconfig.node.base.json` and one `tsconfig.web.base.json` at the root.

**5. Low – Package metadata drift.**

- `apps/api/package.json:45-47` has `"files": ["src"]` but its `exports` point to `dist`.
- The extension uses `vite@^8` and `@types/node@^25`, plex uses `^7` and `^24`.
- `dotenv` is a runtime dependency of a browser extension.
- `video.js` is a devDependency but is imported at runtime.
- `ENV APP_VERSION` in `apps/server/Dockerfile:66-67` is never read.
- The extension's dev proxy targets `:4000` while the API listens on `3113`.

**6. Low – Mixed import specifiers.**

- Plex mixes `.mts` and `.mjs` in the same file (`main.mts:13-16`, `media.store.mts:3-11`).
- `apps/api/src/types/fastifyServices.d.ts` imports `.mts` while the rest of the API uses `.mjs`.
- There are two type roots (`api/src/types/` and `api/src/shared/types/`), and the module augmentations live in hand-written `.d.ts` files.

#### Type safety

**7. High – Hand-maintained duplicate of the media API contract in plex, with confirmed drift.**

- Confirmed mismatches:
    - `Media.overview`, `backdropPath` and `posterPath` are `string` in `plex/src/types/media.type.mts:6-9` but nullable in `index.schema.mts:16-19`.
    - `GenreWithPoster.id: number` vs. `id: z.string()`.
    - `UserSettings.display.availabilityLabel?` is optional in plex but always set by `.default()` in the API.
    - `RequestMediaResponse.mediaAvailability?: number` vs. `z.enum(MediaAvailability)`.
- Enums are copied as well: `MediaType`, `DiscoverType` and `MediaStatus` (with the typo `UKNOWN`). `RequestStatus` exists three times, and the SSE payload is typed twice.
- Fix: move the contract schemas and enums to `apps/shared/src/contracts/media/*.mts`, export `type X = z.infer<typeof XSchema>`, use them in both apps, and delete the plex copies.

**8. High – Upstream JSON is cast, not validated.**

- Where: `api/src/shared/api.client.mts:43` (`as T`), `:36` (`{ success: true } as T`), `tmdb.client.mts:56`, `jellyfin.client.mts:49-52` (a generic used only as a hidden cast).
- Problem: Jellyseerr's raw payload is typed as our outbound `PaginatedResponse`. When upstream changes a field, the failure shows up as a confusing serializer 500.
- Fix:

    Keep separate upstream DTO schemas and map them to the contract explicitly.

**9. High – Plex and the extension trust every response blindly.**

- Where: `plex/src/utils/fetchJson.mts:17` (`as T`, usually inferred from the caller), `ingest.api.mts:25,29`, `ingest.store.mts:28-31`, `commentPicker.mts:217`, `options/storage.mts:5,9`.
- Fix: use `fetchJson(schema, url, init)`, and validate `chrome.storage` with an `OptionsSchema`.

**10. High – Error handler is typed** `any`**, and both branches of its ternary are identical.**

- Where: `api/src/shared/errors/errorHandler.mts:5-23`.
- Problem: `error: isClientError ? error.message : message` gives the same value either way, so raw 5xx messages and pg codes reach clients.
- Fix:

**11. Medium – The** `isIngestJob` **type guard is unsound and hand-rolled.**

- Where: `skeleton.worker.mts:197-222`.
- Problem: it never checks `submittedTags` and duplicates checks that `IngestPayloadSchema` already does.
- Fix: define `IngestJobSchema` in shared and derive `IngestJob` with `z.infer`.

**12. Medium – Repository return types don't match the SQL.**

- `resource.repo.mts:49-53` selects 2 columns but returns the full `ResourceRow`.
- `binkiPlex.repo.mts:191-194` uses `SELECT *`.
- `tags.repo.mts:5` has no type argument, so `rows[0].id` is `any`.
- `type: string` columns force runtime guards, and `outcome: string | null` should be a union.
- Fix: use `Pick<>`, list columns explicitly, use literal unions, and give every `query<>()` a type argument.

**13. Medium – Enums are erased to** `string` **at layer boundaries.**

- `BodyUpdateUserSchema.status: z.string()` leads to `as UserMediaStatus` (`binkiPlex.service.mts:554`).
- `MediaSchema.mediaType: z.string()`.
- `BLOCKED_REQUEST_STATUSES` is a `Set<string>`.
- `jellyseerrUserId` is a `string` in one schema and a `number` in another.
- Fix: use `z.enum(UserMediaStatus)` / `z.enum(MediaType)` end to end.

**14. Medium – Query strings are parsed by hand.**

- `page: z.string().default('1')` followed by `Number(...)`, and `includePosters === 'true'`.
- Fix: `z.coerce.number().int().positive().default(1)` and `z.stringbool()`.

**15. Medium – Invalid stored settings are returned anyway.**

- Where: `binkiPlex.service.mts:98` (`parsed.success ? parsed.data : (settings as UserSettings)`).
- Problem: the response serializer rejects the invalid object, so the user gets a 500.
- Fix: return `UserSettingsSchema.parse({})` and log the failure.

**16. Medium – Non-null assertions on values that can be missing.**

- `process.env.X!` in `postgres.infra.mts:145-146`, `jellyfin.client.mts:320-321`, `jellyseerr.client.mts:38-39`, `s3ObjectStorage.infra.mts:60-68` and `skeleton.worker.mts:24,56,68`.
- `plex/src/store/auth.store.mts:12-13`: `user.value!.id` throws when logged out, which makes the guard at `DiscoverBrowse.vue:106` dead code.
- Also `optionsController.mts:34`, `jellyseer.service.mts:28`, and eight `discoverType.value!` in `DiscoverBrowse.vue`.
- Fix: one Zod env `config.mts` (there are 59 `process.env` reads today), `computed(() => user.value?.id)`, and `Record<DiscoverType, …>` instead of `Map`.

**17. Low – Other** `as` **casts that can be removed.**

- `media.store.mts:90-98` (`as MediaCacheKey`).
- `binkiPlex.service.mts:118,125`.
- `features.mts:25`.
- `Watch.vue:74` (`as unknown as`).
- `commentPicker.mts:283`: use `declare global`.
- `contextMenu.mts:25-28`: write one type guard.
- `optionsController.mts:158-161`: narrows `unknown` to `any[]` without checking.

#### Clean code

**18. Medium – Dead code and unused exports.**

- API: `removeUserMediaFromList`, `purgePortalMediaAfterServerDelete`, `JellyseerrService.getMediaByType`, the whole `tmdb.client.mts`, `getItemsByIds`, `insertTvSubmission`, `MappedMediaData`, `JellyfinLibraryQuery`, `ParamsPortalRequest`.
- Plex: `QuickActionButtons.vue` (contains `console.error(1)`), `MediaOnServer`, `CollectionType`, `personalKeys`/`getPersonalMedia`, the no-op `catch` in `fetchJson.mts`, a debug `console.error` in `SettingsPopUp.vue:75`.
- Duplicate SSE endpoint `/api/portal/events`.
- Fix: delete them. `knip` would find the rest.

**19. Medium – Settings edits mutate the default constant.**

- Where: `settingsServer.store.mts:9,53` (`ref(DEFAULT_SERVER_SETTINGS)`).
- Fix: `ref(structuredClone(DEFAULT_SERVER_SETTINGS))`, or a factory function.

**20. Medium – Project conventions are broken.**

- Types in implementation files: `availability.mts:6-11`, `usePortalEvents.mts:10-17`, inline row types in `binkiPlex.repo.mts:20-25`.
- Runtime values in `types/`: `ingest-extension/src/types/options.type.mts:8-44`.
- Module constants outside `constants.mts`: `skeleton.worker.mts`, `pendingIngest.sweeper.mts`, `jellyfin.service.mts:19-20` (a local `LIBRARY_PAGE_SIZE = 200` shadows the exported `24`), `jellyfin.client.mts`, `s3ObjectStorage.infra.mts`, `dependencyHealth.mts`, `binkiPlex.service.mts:493-494`, `optionsController.mts:120`.

**21. Low – Naming and typos.**

- Typos: `plexPostgress`/`ingestPostgress`, `MEDIATYPE_TO_JELLFIN_TYPE`, `DISCOVER_TYPE_TO_JELLSEERR_TYPE`, `CATERGORY_META`, `JellysseerId`, `MediaStatus.UKNOWN`, `handleUnhandleRejections`, `"Sageguard"`.
- Misleading names: `jellyseer.service.mts`, and `handleViteErrors` (it handles Vue errors).
- Leftover `// *****` dividers.
- `console.*` used in 8 API files next to `logger`.

**22. Low – Errors are stringified instead of chained.**

- `throw new ExtendedError(`…: ${e}`)` (`binkiPlex.service.mts:575,635`) loses the stack.
- Fix: pass `{ cause: e }`.

#### Design patterns

**23. Medium –** `Map`**s keyed by enums lose exhaustiveness checks.**

- `processorRegistry.mts:26` (`Map<string, …>`, plus a runtime "no processor" branch) and `media-portal/constants.mts:20-29`.
- Fix: `Record<IngestResourceType, IngestProcessorFn>`. Also unify `IngestPageKind` and `IngestResourceType` as one `as const` object in shared.

**24. Medium – Requests bypass the service layer and the flows are copy-pasted.**

- `requestMediaThroughPortal` (`binkiPlex.service.mts:580-641`) calls `jsClient` directly and repeats the claim → request → wait → rollback sequence of `handleTvRequest`/`handleMovieRequest`. The service is 790 lines.
- Fix: extract a `RequestCoordinator` with one `submit()` method, and split out the catalog and webhook services.

**25. Medium – The extension background caches stale options.**

- Where: `background.mts:5-10`, `optionsController.mts:13-16`.
- Problem: listeners run before `loadOptions()` resolves, and nothing listens to `storage.onChanged`.
- Fix: await the load inside `runIngestFlow` and reload on `onChanged`.

**26. Low – Messages and options lack discriminated unions and schemas.**

- `CommentPickerMessage` should be a union of three object types.
- `messaging.mts:6-10` never validates `payload`.
- `options.type.mts` is an over-engineered mapped type.
- Fix: one `OptionsSchema` with `z.infer`. It replaces most of the roughly 100 lines of `normalize*` code.

**27. Low – Static-only classes used as namespaces.**

- `plex/src/utils/apiClient.mts`, `plex/src/errors/errorHandler.mts` (needs `.bind()`), `api/src/shared/errors/errorHandler.mts`.
- Fix: plain module functions.

**28. Low – Same host lists in four places.**

- `ingestUrlPolicy.mts`, the extension's `content/constants.mts`, `utils/constants.mts` and `urlTagSuggestions.mts`. The extension misses `youtu.be` and `redd.it`.
- Fix: export `ingestPageKindForHost` from shared and use it everywhere.

**29. Low – Route generics and response schemas are inconsistent.**

- Explicit `api.get<{ Params… }>` alongside `ZodTypeProvider`.
- Playback, episodes, webhooks and `/api/ingest` have no `response` schema.

#### Performance

**30. Medium – Unbounded catalog-sync fan-out.**

- Where: `jellyfin.service.mts:70-76`.
- Problem: `Promise.all(items.map(hydrateItem))` sends one request per library item.
- Fix: cap concurrency, or skip hydration entirely.

**31. Medium – N+1 sequential queries.**

- `binkiPlex.repo.mts:37-49`, `claimTvSeasons`, `tags.repo.mts`, and the per-season `deleteSeerrRequest` loops.
- Fix: batch with `unnest` / `= ANY($n)`.

**32. Medium – Genre posters fetched one at a time.**

- Where: `binkiPlex.service.mts:124-156`.
- Problem: roughly 2 × 20 serial upstream calls on every Browse load.
- Fix: `Promise.all` plus a short cache.

**33. Medium – Plex bundle size.**

- Vuetify star imports cancel tree-shaking, `@mdi/font` is imported twice, and all routes are eager.

**34. Low – Small leaks in the API process.**

- `JOB_METRICS.durationMs` grows forever.
- `ApiStatsSerializer` uses sync fs and a non-unref'd interval.
- `waitForDownload` holds HTTP requests open for 30 s.

### What's done well

- `strict` everywhere, effectively one `any`, consistent `catch (err: unknown)` with `instanceof` narrowing, and `import type` throughout.
- `IngestPayloadSchema` / `CapturedCommentSchema` in `apps/shared` are a real single source of truth. This is the pattern to copy for the media contract.
- `fastify-type-provider-zod` validates inbound data, and most portal routes have response schemas. Webhook bodies are parsed from `unknown`.
- Good discriminated unions and exhaustive `Record<…>` maps (`PortalSseEvent`, `IngestMessageResponse`, `Record<ContextMenuId, …>`).
- Architecture boundaries are enforced by `no-restricted-imports`, and feature gating is clean and well typed.

### Top 5 actions

1. A `typecheck` Turbo task in CI, plus type-aware ESLint that also covers `apps/server`.
2. Move the media-portal schemas and enums into `apps/shared` and derive the frontend types with `z.infer`.
3. Give the HTTP clients a schema parameter, and separate upstream DTOs from contract schemas.
4. `noUncheckedIndexedAccess` and friends, plus one Zod-validated `config.mts`.
5. Fix these concrete bugs, then batch the N+1 queries and cap the fan-out:
    - the error-handler ternary;
    - the `DEFAULT_SERVER_SETTINGS` mutation;
    - `auth.userId!`;
    - the `as UserSettings` fallback;
    - stale extension options.

---

## Senior Vue Developer

**Summary.** Both apps use modern Vue throughout: `<script setup>`, typed `defineProps`/`defineEmits`/`defineModel`, setup-style Pinia stores, and a normalized media cache keyed by `type:id`. Accessibility and the async-state UX are clearly deliberate. The main problems are in data flow:

- Every `MediaCard` makes its own `/media-info` request (an N+1 pattern).
- Cache invalidation is missing.
- Settings edits write directly into a shared constant.
- An aborted search shows an error.

Bundle size is much larger than it needs to be. In the extension, the service worker can run with unloaded or stale options, and the context-menu flow crashes because the `notifications` permission is missing.

### Findings

**1. Critical · Performance: every rendered card fetches its own media details (N+1)**

- Where: `apps/plex/src/components/mediaCard/MediaCard.vue:67-76`. The `watch(() => props.media, …, { immediate: true })` calls `mediaStore.selectMedia`, which calls `hydrateMedia` and then `ApiClient.fetchMediaInfo`.
- Problem: Browse makes about 60 requests on first load, and DiscoverBrowse makes 160+ after prefetching. Each request costs a Postgres query plus a Jellyseerr call (`binkiPlex.service.mts:200-241`). When the backend has trouble, each failure becomes its own snackbar.
- Fix: list endpoints already return enough to draw a card. Hydrate only when the dialog opens (`RequestDialog.vue:410` already does this), or batch with one `POST /media-info/batch` per page.

**2. High · UX: fast typing shows a "Search failed" error**

- Where: `apiClient.mts:62-66` aborts the previous search, `fetchJson.mts:5-9` re-throws the `AbortError`, and `Browse.vue:225-230` shows an error snackbar. `AppBar.vue:93-95` emits on every keystroke with no debounce.
- Fix:

    Add a 250–300 ms debounce and delete the dead `try/catch` in `fetchJson`.

**3. High · Reactivity / Architecture: settings are edited in place, defaults get overwritten, and caches are never invalidated**

- Where: `settingsServer.store.mts:9,53`, `SettingsPopUp.vue:50-78`.
- Problems:
    - `ref(DEFAULT_SERVER_SETTINGS)` lets the reactive proxy write into the constant itself.
    - There's no draft copy, so toggles apply immediately, even without Save.
    - The dialog never handles `update:model-value`, so Esc and backdrop clicks do nothing.
    - `onSettingsSaved` is emitted before `await save()`, with no `try/catch`.
    - A debug `console.error` is left in.
    - Changing `includeLanguages` doesn't reset the media store (`media.store.mts:40` returns early once `loaded` is true).
- Fix: `structuredClone` the defaults, add a local draft and a `save(draft)` action that calls `mediaStore.reset()` when the languages change, use `defineModel<boolean>()`, and add a Cancel button.

**4. High · Performance: Vuetify tree-shaking is disabled, routes are eager, and video.js and the MDI font always load**

- Where: `plugins/vuetify.mts:4-5,9-10` (`import * as components/directives`) cancels `vite-plugin-vuetify({ autoImport: true })`. `plugins/router.mts:3-8` loads every route eagerly, so video.js and `video-js.css` ship in the main chunk. `@mdi/font` is imported twice (`main.mts:8`, `vuetify.mts:3`).
- Unused dependencies: `@fontsource/noto-sans`, `@fontsource/roboto`, `lodash-es`, `material-letter-icons`. `video.js` is in `devDependencies`.
- Fix:

    Consider `@mdi/js` with the `mdi-svg` icon set, as the extension already does.

**5. High · Architecture: DiscoverBrowse wipes the collection that Browse's rows share**

- Where: `DiscoverBrowse.vue:105-121` calls `resetDiscoverCollection` on every mount and prefetches up to 12 pages.
- Problem: when you come back to Browse, the Trending row renders 200+ cards in one non-virtualized row, and the hero can switch to a different title. Nothing cancels the prefetch loop after navigation.
- Fix: don't reset a collection that already has items, `.slice(0, ROW_LIMIT)` the Browse rows, and cancel the loop in `onUnmounted`.

**6. Medium · Reactivity: cache merges can't clear fields**

- Where: `media.store.mts:105` does `Object.assign(existing.media, media)`.
- Problem: `JSON.stringify` drops `undefined`, so fields like `jellyfinId` and `requestedBy` never clear. SSE refreshes (`usePortalEvents.mts:70`) also leave out `userId`, so `onWatchlist` is never refreshed.
- Fix: replace the cached object (`existing.media = media`), or send explicit `null` from the API. Pass `auth.user?.id` in the SSE refresh.

**7. Medium · Performance: large lists are deeply reactive and never trimmed**

- Where: `media.store.mts:21`, `Library.vue:45,80`, `Request.vue:46`.
- Problem: every `Media` object is deeply proxied, the cache only grows, and `MediaGrid` has no virtualization.
- Fix: use `shallowRef` for the lists, `markRaw` media in the cache, and `content-visibility: auto; contain-intrinsic-size: 150px 260px;` on the grid items (or `v-virtual-scroll`).

**8. Medium · Architecture: RequestDialog is 841 lines and edits store data directly**

- Where: `RequestDialog.vue`.
- Problems:
    - It changes cached objects directly (`:484-503`, `:538`).
    - The watcher at `:397-420` has no `catch` and no stale-request token.
    - `function watch()` (`:515`) shadows the Vue import.
    - There's a nested ternary in the template.
- Fix: split it into `MediaDetailsHeader`, `EpisodeList`, `SeasonRequestGrid` and a `useMediaRequest` composable. Add store actions `markRequested` and `markOnWatchlist`, and rename `watch()` to `onWatchNow()`.

**9. Medium · Reactivity: Watch.vue playback loading can race**

- Where: `Watch.vue:100-130`.
- Problem: no request token, and every call adds another `one('loadedmetadata')` handler, so an older load can apply the wrong resume position. The height is hard-coded as `calc(100dvh - 64px)`.
- Fix: guard with a `loadToken` and remove the previous listener with `off()`.

**10. Medium · Over-engineering: ScrollableGroup's custom scrolling conflicts with CSS**

- Where: `ScrollableGroup.vue:66-93,172`.
- Problem: a rAF loop and `scroll-behavior: smooth` fight over the same scroll. Arrow state goes stale on swipe or late-loading content, the listener isn't passive, there's commented-out code at `:103`, and `onTitleClicked` is a function prop.
- Fix:

    Use `emit('seeAll')` instead of the function prop.

**11. Medium · Performance: genre cards load 40 full-size images up front**

- Where: `Browse.vue:245-248`, `getImages.mts:30`.
- Problem: `w1280_filter(duotone…)` CSS backgrounds aren't lazy-loaded. Also, `getImageURL_300` only checks for `undefined`, but TMDB returns `null`, producing `…/w300null` requests.
- Fix: use `w780` or `w300` with `<img loading="lazy">` and a CSS duotone overlay, and check `if (!path)`.

**12. Medium · a11y: genre cards are buttons that do nothing**

- Where: `Browse.vue:61,79`.
- Problem: 40 focusable `<button>` elements have no `@click`.
- Fix: link them to a genre route, or render them as `<div>`.

**13. Medium · Architecture: profile reset logic is copied in three places**

- Where: `ProfileSelection.vue:58-65`, `useSwitchProfile.mts:12-17`, `Browse.vue:184-186`. Also `auth.userId` uses `!`, so the `=== undefined` checks in `DiscoverBrowse`, `Library` and `Request` can't succeed.
- Fix: add an auth store action `switchTo(user?)`, and use `computed(() => user.value?.id)`.

**14. Medium · UX: the global error handler shows raw internal errors**

- Where: `errors/errorHandler.mts:5-16`.
- Problem: snackbars like "Unhandled promise: HTTP 500 …".
- Fix: log the details, show one generic rate-limited message, filter out `AbortError`, and rename `handleViteErrors` to `handleVueError`.

**15. Medium · Clean code: dead code and leftovers**

- `Collection.vue` (unused, has a `Test` label and a hard-coded TMDB URL).
- `QuickActionButtons.vue`.
- `MediaGrid.vue:23-40` listens for events that `MediaCard` never emits.
- `DiscoverBrowse.vue` has no-op handlers and unused CSS.
- `personalKeys`/`getPersonalMedia` and the `selectMedia` alias.
- `getImageURL_og` and the `getGenreColorTone` tables.

**16. Low · Clean code: convention violations**

- Types outside `types/`: `usePortalEvents.mts:10-17`, `availability.mts:6-11`.
- Constants outside `constants.mts`: `Browse.vue:128-129`, `DiscoverBrowse.vue:64-68,124`, `ScrollableGroup.vue:47-48`, `SideBar.vue:48,54`, `discoverCatalog.mts:3-9`, `ingest-extension/src/options/App.vue:73-97,113-114`, `ingest.store.mts:7`.

**17. Low · Design patterns: store and API-client details**

- `initialize()` waits for the discover requests before loading genres; run them with `Promise.all`.
- Discover state is read through methods instead of getters.
- The search `AbortController` is static state in `ApiClient`; let callers pass an `AbortSignal`.

**18. High · Architecture (extension): the service worker's options cache may be unloaded and is never refreshed**

- Where: `background.mts:8-10`, `ingestFlow.mts:6`, `ingest.api.mts:6,15`, `optionsController.mts:13-16`.
- Problems:
    - After an MV3 wake-up, the handler can run with the default `apiUrl` and no Cloudflare headers.
    - Changes on the options page never reach the worker.
    - Theme sync in `AppBar.vue:38-49` reads the stale cache.
    - `updateOptions` writes a full snapshot, which can overwrite newer settings.
- Fix:

    Persist only the patched keys.

**19. High · Architecture (extension): context-menu ingest crashes and the badge count gets stuck**

- Where: `contextMenu.mts:36-50`, `manifest.json:5`.
- Problem: `chrome.notifications` is used without the `notifications` permission, so it throws in both the `try` and the `catch` and `decrementBadge()` never runs. `icon.png` is missing, and the badge count resets when the worker restarts.
- Fix: add the `notifications` permission and ship the icon. Move `decrementBadge()` into `finally`, and store the count in `chrome.storage.session`. Consider dropping `tabs`.

**20. Medium · Reactivity (extension): the options page's pinned save buttons never update**

- Where: `options/App.vue:163-176`. The `scrollTick` dependency is commented out at `:164`.
- Fix: restore `void scrollTick.value;`, or use an `IntersectionObserver`.

**21. Low · Clean code (extension): duplicated helpers and unnecessary deep watchers**

- `tagMeta`/`metaFor` is copied three times (`TagCatalogTree.vue`, `TagLibraryRowsSettings.component.vue`, `DomainRule.component.vue`).
- `childrenOf(cat)` is recomputed on every render.
- `deep: true` is set on arrays that are always replaced.
- `DomainRule.component.vue:12` uses `:key="index"` on a list where rows can be deleted.

### What's done well

- The normalized media cache uses `MediaCacheKey` and deduplicates in-flight hydration through `hydrationPromises`. Discover pages are de-duplicated.
- Player lifecycle: `shallowRef` for video.js, composables with explicit `bind`/`stop`, a `pagehide` keepalive report, and a progress queue that finishes before Continue Watching refetches.
- Request tokens prevent stale responses in `Library.vue`, `RequestDialog.loadEpisodes` and `Request.fetchRequests`.
- Accessibility is above average: real `<button>` cards, `aria-pressed`, `role="status"`/`alert`, `prefers-reduced-motion` and `focus-visible` styles.
- `MediaGrid` is a generic component with typed slots.
- The extension uses tree-shaken `@mdi/js` icons, guarded messaging, and a content script that cleans up after itself.

### Top 5 actions

1. Remove per-card hydration in `MediaCard.vue:67-76`.
2. Fix the settings flow: clone the defaults, edit a draft, await the save, make the dialog dismissible, and invalidate the cache when languages change.
3. Shrink the bundle: no Vuetify star imports, lazy routes, one icon strategy, and remove unused dependencies.
4. Fix search: ignore `AbortError`, debounce, and simplify `fetchJson`.
5. Make the extension service worker reliable: await options, react to `onChanged`, write only patched keys, add the `notifications` permission, and decrement the badge in `finally`.

---

## Security Engineer

### Threat model

Binki runs on the owner's home server and can only be reached through WireGuard. Its users are the owner plus a few trusted household members and friends. With no public exposure, the lack of authentication and rate limiting is treated as a deliberate trade-off, not a Critical issue.

Four attackers are still in scope:

- **A compromised or careless VPN peer.** For example, a friend's laptop with malware, or a kid on the household profile.
- **A malicious URL submitted for ingest.** Especially a Reddit post the attacker controls, which is fetched and parsed and passed to yt-dlp and ffmpeg.
- **A malicious web page or browser extension in a VPN user's browser.** This covers DNS rebinding, CORS abuse, and attacks on the ingest extension.
- **A compromised dependency or container image.** npm, PyPI (yt-dlp), Docker Hub, GitHub Actions.

### Summary

The core ingest SSRF controls are good:

- Our own fetches use a strict https-only host allowlist with manual redirect re-checks.
- Arbitrary generic URLs are stored as metadata only and never fetched.
- yt-dlp only receives YouTube, TikTok or Reddit page URLs, placed after `--`.
- SQL is fully parameterized.
- Jellyfin, Jellyseerr and TMDB keys never reach the browser.

The main gap is yt-dlp itself. It follows whatever a Reddit post embeds, through its generic extractor, with no extractor allowlist and no egress isolation. From inside the compose network that means it can reach Postgres, RabbitMQ, RustFS, Jellyfin, Jellyseerr and the LAN.

Secondary issues:

- No Host header check, combined with a CORS rule that trusts every `chrome-extension://` origin. This allows DNS rebinding and lets any installed extension use the API.
- `.yarnrc.yml` disables TLS verification for package installs.
- Containers run as root while ffmpeg parses untrusted media.
- Infrastructure credentials have more privilege than they need.

No secrets are committed. `.env*` files are gitignored, absent from git history, and not copied into the images.

### Findings

#### F1. yt-dlp SSRF through Reddit embeds and the generic extractor

- **Severity (in context):** Medium. **If exposed publicly:** High. **Category:** SSRF.
- **Where:**
    - `ytdlpDownload.mts:138-141`: no `--ies`, no `--ignore-config`, no check of the post-extraction URL.
    - `redditPostIngest.mts:76-77`: `rich:video` posts are classified as video.
    - `redditPostIngest.mts:279-285`: the permalink is handed to yt-dlp.
    - `ytdlpPreview.routes.mts:23`: the same path through the preview route.
    - `docker-compose.dev.yml:5-24`: `server` shares the network with every dependency.
- **Attack:**
    1. An attacker makes a Reddit link post to a page they control (oEmbed or `og:video`).
    2. Someone ingests it. The URL passes `isAllowedIngestPageUrl` because the host is `reddit.com`.
    3. yt-dlp's Reddit extractor returns `url_transparent`, and the generic extractor follows redirects or `og:video`/m3u8 references to places like `http://jellyfin:8096/…`, `http://rabbitmq:15672/` or `http://192.168.1.1/`.
    4. Blind GET actions on LAN devices are possible, and responses that look like media get stored in S3.

        Confirm with `yt-dlp -v --dump-single-json <rich:video post>`.
- **Fix:**
    - `--ignore-config --ies "youtube,youtube:tab,tiktok,reddit"` (or `"default,-generic"`).
    - Reject the job unless `extractor_key` is in `{Youtube, TikTok, Reddit}` and `webpage_url`/`original_url` pass the allowlist, then download with `--load-info-json`.
    - Put the worker on an egress-only network, add `DOCKER-USER` rules that drop traffic to RFC1918, 169.254/16 and the WireGuard subnet, or use a filtering `--proxy`.

#### F2. DNS rebinding and over-broad CORS

- **Severity (in context):** Medium. **If exposed publicly:** High. **Category:** CORS / browser-to-private-network.
- **Where:**
    - `corsOrigin.mts:4`: any `chrome-extension://` origin is allowed.
    - `binkiApiPlugin.mts:94-103`: requests with no Origin are allowed.
    - `apps/server/src/index.mts:8-11`: no Host check.
    - `apps/api/src/constants.mts:2`: binds `0.0.0.0`.
    - `binkiPlex.routes.mts:364-374`: SSE uses the same rule.
- **Attack:** a malicious page rebinds its DNS to the WireGuard IP on port 3113 and becomes same-origin. From there it can call every endpoint: read users, change settings, request media as any user, trigger ingests and stream the library out. Separately, any installed extension can make credentialed calls.
- **Fix:** an `onRequest` hook that rejects requests whose `request.hostname` isn't in `BINKI_ALLOWED_HOSTS` (421/403), and a reverse-proxy `default_server` that drops unknown hosts. Pin the extension origin to `chrome-extension://<your-id>` via `BINKI_EXTENSION_ORIGIN`.

#### F3. Profile identity is client-side only; the Jellyfin admin key bypasses per-user restrictions

- **Severity (in context):** Low (household trust is the design). **If exposed publicly:** Critical. **Category:** AuthN/AuthZ.
- **Where:**
    - `auth.store.mts:15-21,25` (user stored in localStorage) and `useSwitchProfile.mts:12-17`.
    - `userId` is trusted from params, query or body (`binkiPlex.routes.mts:75-101,312-316,331-343,345-362,400-433`, `portal.routes.mts:28-35`).
    - `jellyseerrUserId` comes from the client (`index.schema.mts:187`, `binkiplex.schema.mts:30`, `binkiPlex.service.mts:599-604`).
    - The Jellyfin admin key is used for all calls (`jellyfin.client.mts:320-322`).
- **Attack:** a kid's profile can stream anything (Jellyfin parental ratings aren't enforced). Anyone can file auto-approved Jellyseerr requests as the admin, or overwrite another profile's state.
- **Fix (optional):** a per-profile PIN that issues an HttpOnly signed session cookie, with `userId` and `jellyseerrUserId` derived from the session. Per-user Jellyfin filtering or tokens for restricted profiles.

#### F4. TLS verification disabled for all package installs

- **Severity (in context):** Medium. **If exposed publicly:** Medium. **Category:** Supply chain.
- **Where:** `.yarnrc.yml:1` (`enableStrictSsl: false`), copied into both Dockerfiles (`:12`) and used in CI (`build-lint.yml:46`). The `@binki`/`@bp` scopes route to Verdaccio with `npmAlwaysAuth`.
- **Attack:** an on-path attacker serves tampered tarballs during `yarn add`/`yarn up`, or steals the Verdaccio token. Lockfile checksums partly mitigate this, but only for existing dependencies.
- **Fix:** remove the line. Use `caFilePath`/`httpsCaFilePath` for a private CA.

#### F5. Containers run as root while ffmpeg and yt-dlp parse untrusted media; no resource caps

- **Severity (in context):** Medium. **If exposed publicly:** High. **Category:** Container hardening.
- **Where:** `apps/server/Dockerfile:62-88` and `apps/api/Dockerfile:47-73` (no `USER`). `docker-compose.dev.yml:6-24` has no `read_only`, `cap_drop`, `security_opt` or limits.
- **Attack:** an ffmpeg or yt-dlp parser bug gives root in the container. The attacker reads every secret in the env and pivots over the compose network.
- **Fix:** `USER node`, `read_only: true`, `tmpfs: /tmp:size=20g` (or a volume), `cap_drop: [ALL]`, `security_opt: [no-new-privileges:true]`, `mem_limit`, `pids_limit`.

#### F6. Over-privileged infrastructure credentials; Valkey has no password

- **Severity (in context):** Low. **If exposed publicly:** High. **Category:** Least privilege.
- **Where:**
    - Both Postgres URLs use the superuser (`postgres.infra.mts:76` runs `CREATE EXTENSION`).
    - RabbitMQ uses the admin user.
    - `docker-compose.deps.yml:31` starts Valkey with no `--requirepass` even though `VALKEY_URL` contains credentials. Either Valkey is open to the network, or AUTH fails and dedupe is silently off (`ingestLock.mts:24`). **Verify which.**
- **Fix:** per-database roles (`binki_plex`, `binki_ingest`), with `pgcrypto` installed in the init script. A vhost-scoped RabbitMQ user. `--requirepass ${VALKEY_PASSWORD}`, with the healthcheck updated to match.

#### F7. Unbounded or expensive ingest resources (DoS)

- **Severity (in context):** Low. **If exposed publicly:** High. **Category:** DoS.
- **Where:**
    - 8 GiB per video (`ingestLimits.mts:10`), 30-minute timeout.
    - Temp files in `os.tmpdir()`, the container's writable layer.
    - No `!is_live` filter, so live streams download until the timeout.
    - `MAX_RETRIES = 5`.
    - One global in-memory quota.
    - The preview route skips the quota and has no concurrency cap.
    - The `tags` array is unbounded, and so are SSE subscriber counts.
- **Fix:**
    - A cap of about 2 GiB and `--match-filter "!is_live & !was_live"`.
    - A size-capped temp volume.
    - Fail fast without retry on errors like "too large".
    - `tags.max(32)` with each string `.max(64)`.
    - Keep the preview route dev-only behind a semaphore.
    - Cap SSE clients.

#### F8. Secrets and spoofable data in logs

- **Severity (in context):** Low. **If exposed publicly:** Medium. **Category:** Logging.
- **Where:**
    - The webhook token is accepted as `?token=` (`portalHooks.mts:123-126`), and full URLs are logged (`requestLog.mts:61`, `errorHandler.mts:15`).
    - `requestLog.mts:16-26` trusts `X-Forwarded-For` even when `BINKI_TRUST_PROXY=0`.
    - Stack traces are returned when `NODE_ENV=development`.
- **Fix:** accept the token only in the `X-Binki-Webhook-Token` header (or redact it from logs), and use `request.ip`.

#### F9. Scraped content stored raw; `javascript:` permalinks accepted (latent XSS)

- **Severity (in context):** Info. **If exposed publicly:** Medium. **Category:** Stored XSS (latent).
- **Where:** `ingest.contract.mts:20` (`z.string().url()` accepts `javascript:`). Reddit text is stored verbatim. There is no `v-html` anywhere today.
- **Fix:** restrict `permalink` with `isAllowedIngestPageUrl`. Any future UI should use `{{ }}` and only allow `https:` in `:href`.

#### F10. Extension: Cloudflare Access secret in plaintext storage, sent to any API URL; handlers don't check the sender

- **Severity (in context):** Low. **If exposed publicly:** Medium. **Category:** Browser extension.
- **Where:** `optionsController.mts:112-121`, `options/storage.mts:5-21`, `ingest.api.mts:6-22`, `messaging.mts:5`, `content/commentPicker.mts:288`, `manifest.json:5-6`.
- **Note:** a Cloudflare Access client suggests you may have, or plan, a Cloudflare Tunnel. **If so, the WireGuard-only model doesn't hold, and F1, F2, F3 and F7 move to their "If exposed publicly" ratings.**
- **Fix:** only attach CF headers when the `apiUrl` origin matches the configured CF hostname. Add `if (sender.id !== chrome.runtime.id) return;` to both handlers, and drop `tabs` if `activeTab` is enough.

#### F11. Unpinned images, actions and yt-dlp

- **Severity (in context):** Low. **If exposed publicly:** Low. **Category:** Supply chain.
- **Where:** `rustfs:latest`, `jellyfin/jellyfin` (no tag), `jellyseerr:latest`, `node:22-bookworm-slim` with no digest, unpinned `pip install yt-dlp`, and actions pinned to tags rather than SHAs.
- **Fix:** pin images by digest (with Renovate), pin actions to SHAs, and pin yt-dlp with `--require-hashes` on a bump schedule.

#### F12. Ingest URL policy is hostname-only (Info)

- **Severity (in context):** Info. **If exposed publicly:** Info. **Category:** SSRF design note.
- **Where:** `ingestUrlPolicy.mts:12-41,85-100`. No DNS or IP check.
- **Impact:** not exploitable today, because generic URLs are never fetched.
- **Fix (future):** if a feature ever fetches generic URLs, resolve DNS and pin the IP, block private, loopback, link-local and CGNAT ranges, and recheck on every redirect.

### Accepted risks given WireGuard-only deployment

- No authentication on `/api/*`, and no profile PIN (F3), as long as all peers are trusted and no profile needs parental controls.
- No per-client rate limiting. The global 30/min ingest quota is enough for a household.
- `/health`, `/metrics/jobs` and the status page are unauthenticated. They only expose versions and queue depths.
- SSE endpoints are open to all peers.
- The Jellyfin image proxy is unauthenticated (its params are enum or regex-restricted).
- The RabbitMQ management UI and RustFS console are bound to `127.0.0.1`.
- The CF secret sits in `chrome.storage.local` on your own devices.

### What's done well

- The HTTP fetcher is https only, with no userinfo or port, IP literals and local suffixes rejected, manual redirects with a per-hop allowlist, a streamed byte cap and a timeout (`httpDownload.mts:66-167`).
- yt-dlp runs through `spawn` with an argv array, the URL after `--`, a host gate, stdout caps, timeouts and tree kill.
- Arbitrary hosts are metadata-only, and the worker re-validates jobs as it dequeues them.
- Storage keys are server-generated, so S3 path traversal is impossible.
- SQL is fully parameterized; the only `format()` uses `%I`.
- The HLS proxy rejects encoded traversal, pins the origin, constrains paths and strips `ApiKey` from playlists.
- Jellyseerr inputs are integers or enums.
- API keys stay server-side, and `apps/plex/.env` is not copied into the image.
- The webhook token uses `timingSafeEqual` and fails closed.
- Compose ports are bound to loopback.
- The fonts route uses a hard-coded allowlist.
- Extension host permissions are limited to localhost, there is no `externally_connectable`, and content scripts only use `textContent`.
- Failing CORS rejects the whole request, which also blocks form-POST CSRF.

### Top 5 actions

1. **Lock down yt-dlp (F1):** an extractor allowlist, a check of `extractor_key`/`webpage_url` before downloading, and egress isolation.
2. **Host allowlist plus a pinned extension origin (F2).**
3. **Harden the containers (F5):** `USER node`, read-only, a capped tmpfs, `cap_drop`, `no-new-privileges` and limits, plus `!is_live` and a lower video cap (F7).
4. **Supply chain (F4, F11):** remove `enableStrictSsl: false`, and pin images, actions and yt-dlp.
5. **Least privilege (F6, F8):** per-database Postgres roles, a scoped RabbitMQ user, a Valkey password (check whether dedupe is currently off), and the webhook token only in a header.

---

## Backend API Engineer

**Summary.** For a home deployment the backend is in good shape:

- Dependencies are optional at boot, and features are gated on health.
- Publishing uses confirm channels, and a Postgres-backed pending sweeper covers broker outages.
- Retry counting uses `x-death`.
- Playback is streamed rather than buffered.

The biggest risks are in lifecycle and idempotency:

- SIGTERM is never handled, so `onClose` is dead code in Docker.
- Ingest retries re-upload media under random S3 keys and insert duplicate rows.
- yt-dlp jobs can outlive RabbitMQ's consumer timeout.

On the media-portal side, the problems are:

- Unbounded upstream fan-out.
- A catalog sync that re-runs on every health flap.
- Extra Jellyfin round-trips on every playback range request.
- Handlers that block for 30 seconds while polling.

Route design is also uneven.

### Findings

**1. Critical · Ops/Reliability: no signal handling, so graceful shutdown never runs**

- Where: `apps/api/src/index.mts:14-21`, `apps/server/src/index.mts:33-40`, the Dockerfiles' `CMD ["node", …]`.
- Problem: Node runs as PID 1 with no handlers, so `docker stop` waits 10 seconds and then SIGKILLs. The `onClose` chain (`binkiApiPlugin.mts:83-92`) never runs, and in-flight ingests die mid-upload. Two timers also keep the event loop alive: `requestCount.mts:36-42` (not unref'd) and the HLS idle timers (`jellyfin.client.mts:228-234`).
- Fix:

    Add `init: true` and `stop_grace_period: 30s`, and cancel the consumer and drain in-flight jobs in `onClose`.

**2. High · Reliability/Data: ingest retries aren't idempotent**

- Where: `storageKey.mts:33` (`randomUUID()`), `ytdlpHostedVideoIngest.mts:88-89,100-101`, `redditPostIngest.mts:345-346`, `createBinkiExtensionTables.mts:25-36` (no unique constraint).
- Problem: each retry re-uploads and inserts new rows. Up to 6 copies of multi-GB objects can pile up.
- Fix: deterministic keys, `UNIQUE (resource_id, type)` with an `ON CONFLICT … DO UPDATE` upsert, and skipping steps that are already done.

**3. High · Reliability: long jobs vs. RabbitMQ** `consumer_timeout`**; poison messages loop forever**

- Where: `ingestLimits.mts:14`, `ytdlpDownload.mts:274-276`, `s3ObjectStorage.infra.mts:119-126`.
- Problem: the worst case exceeds RabbitMQ 4.x's 30-minute default timeout, which causes a redelivery and a full re-run. A message that crashes the process (OOM or SIGKILL) gets no `x-death` entry, so it loops forever.
- Fix: use a quorum queue with `x-delivery-limit: 5`, raise `x-consumer-timeout` or keep a job deadline below it, and give `putFile` an `abortSignal`.

**4. High · Reliability: single S3 PUT for up to 8 GiB**

- Where: `s3ObjectStorage.infra.mts:109-129`.
- Problem: a single PUT is capped at 5 GiB, with no part retries and no timeout.
- Fix: `@aws-sdk/lib-storage` `Upload` with `partSize: 64 MiB` and `queueSize: 4`.

**5. High · Performance/Reliability: catalog sync fans out one request per item and re-runs on every health flap**

- Where: `jellyfin.service.mts:70-76`, `binkiApiPlugin.mts:50-59`.
- Problem: `Promise.all` sends one `GET /Items/{id}` per library item, and a single failure fails the whole sync. It's also unnecessary: the sync only needs `Id`, `Name`, `Type` and `ProviderIds`, which `getAllMedia` already returns. On top of that, every offline-to-live transition of `mediaPortal` triggers a full sync.
- Fix: delete `hydrateAllMedia`, and run the sync single-flight and throttled:

**6. High · Performance: extra Jellyfin round-trips on the playback hot path**

- Where: `jellyfin.client.mts:148-161,164-181`, `jellyfin.service.mts:141-145,228-229`.
- Problem: every range request sends a `postBrowserPlaybackInfo` first. `browserTranscodesAudio` starts a session only to `DELETE` it right after.
- Fix: decide direct play vs. HLS once in `getPlaybackInfo`, make `/stream` a pure range proxy, and keep the decision in an LRU cache.

**7. High · API design: request endpoints block for up to 30 seconds**

- Where: `binkiPlex.service.mts:437-438,476-477,605`, `waitForDownload` at `:492-509`.
- Fix: return `202 { outcome: 'checking' }` right away, and broadcast the result over SSE (or rely on the webhook).

**8. High · Performance: unbounded Jellyseerr fan-out and no caching**

- Where: `getRequests` enriches every group with no LIMIT (`binkiPlex.service.mts:265-323`, `binkiPlex.repo.mts:169-189`). `getGenres` makes N sequential calls (`:124-156`). Library pages make 24 concurrent calls.
- Fix: a TTL LRU over `getMediaInfo` that also coalesces in-flight requests, `p-limit(6)`, a paginated `/requests`, and a cache for genre posters.

**9. High · Data layer: catalog sync does one INSERT per row inside a transaction, while a bulk version sits unused**

- Where: `binkiPlex.repo.mts:36-49` vs. the unused `insertMissingBinkiCatalogMedia` at `:467-491`. The same pattern appears in `claimTvSeasons`, `insertTvSubmission`, `replaceJellyfinJellyseerrUserLinks` and `attachTags`.
- Fix:

    Use a single-statement CTE for tags, and cap `tags` in the contract.

**10. Medium · Data layer: a query inside a transaction escapes it**

- Where: `binkiPlex.repo.mts:413-418` calls `insertMovieSubmission`, which uses the pool, not `client`.
- Problem: it can commit even if the transaction rolls back. Latent today, because `recordSubmission` is always false.
- Fix: pass `client` through, or delete the dead branch.

**11. Medium · Data layer: missing and redundant indexes, no migrations**

- Missing FK indexes on `resource_files(resource_id)`, `processing_logs(resource_id)` and `resource_tags(tag_id)`.
- No partial index for the sweeper.
- Redundant: `idx_resources_status` and `idx_user_media_user_id`.
- `idx_user_media_tmdb_id` should be `(tmdb_id, media_type)`.
- `seerr_request` has no primary key, and ingest tables use `TIMESTAMP` without a time zone.
- Schema evolution is ad-hoc `ALTER … IF NOT EXISTS`.
- Fix:

    Adopt a small migration runner such as `node-pg-migrate`, or a `schema_migrations` table.

**12. Medium · Reliability:** `setResourceStatus` **never bumps** `updated_at`**, and stuck** `processing` **rows are never swept**

- Where: `resource.repo.mts:106-117`.
- Fix: `SET status=$2, updated_at=now()`, and have the sweeper reclaim stale `processing` rows.

**13. Medium · Reliability: dead-letter publish isn't confirmed before ack**

- Where: `skeleton.worker.mts:56-60,68-72`.
- Fix: `await channel.waitForConfirms()` before `ack`, or `nack` into a DLX. With quorum queues, this branch goes away.

**14. Medium · Over-engineering: Valkey is optional in the code but required by the feature gate**

- Where: `ingestLock.mts:7-29` (fails open) vs. `features.mts:11`.
- Problem: a Valkey outage disables all ingest. The lock is released without an ownership token, and every error triggers a disconnect.
- Fix: drop `valkey` from the ingest dependencies, or replace the lock with SQL dedupe (`enqueued_at > now() - interval '60s'`). If you keep it, add a token and a compare-and-delete Lua script.

**15. Medium · Reliability: coarse feature gating blocks playback and webhooks**

- Where: `binkiPlex.routes.mts:56`, `portal.routes.mts:8`, `features.mts:10`.
- Problem: a Jellyseerr outage stops video playback, and webhooks get 503s. Jellyfin doesn't retry, so those events are lost.
- Fix: gate per route group. Use `mediaStack` (defined but unused) for playback, and a Postgres-only gate for webhooks.

**16. Medium · API design: route surface is inconsistent, with dead legacy routes**

- Where: `POST …/settings` (should be `PUT`), `POST /api/:type(movie|tv)/update/:userId` (a verb in the path, outside `/api/bp`), a synchronous `POST /catalog/sync`, a no-op Sonarr hook, and `portal.routes.mts:11-35`.
- Problem: the frontend never calls `/api/portal/events`, `/api/user/:userId/media/:type/request`, `/api/:type/update/:userId`, `/api/bp/user/:userId/media`, `/api/bp/catalog/sync` or the Sonarr hook. `requestMediaThroughPortal` duplicates `requestMedia`.
- Fix: delete them. Standardize on `/api/bp/users/:userId/...` and `PUT`, and make catalog sync return 202.

**17. Medium · API design: ingest returns 200 for async work and has no status resource**

- Where: `binkiIngest.routes.mts:11-13`.
- Fix: return `202` with `Location: /api/ingest/:id`, and add a `GET` that returns status plus the last N logs.

**18. Medium · API design: a GET with side effects**

- Where: `binkiPlex.service.mts:196-202`. `GET /media-info` writes `seerr_request.outcome` and broadcasts SSE.
- Fix: move this into the webhook handler or a periodic reconciler.

**19. Medium · API design: validation gaps produce 500s instead of 400s**

- String `page`, string `includePosters`, a hand-parsed `id`, an unbounded `userId`, and a free-form `status` that only fails at the DB CHECK.
- `normalizeUrl` throws a raw `TypeError`.
- `getUserSettings` returns invalid JSON when parsing fails.
- Fix: `z.coerce.*`, `z.enum(UserMediaStatus)`, `.max(64)` on ids, and `ExtendedError(…, 400)` from `normalizeUrl`.

**20. Medium · API design: error handler leaks internals and passes upstream status through**

- Where: `errorHandler.mts:18-23`, `custom.error.mts:4-16`.
- Problem: raw 5xx messages and pg SQLSTATEs reach clients. A bad Jellyfin API key shows up as a 401 from your API. The response envelopes are inconsistent.
- Fix: map upstream 401/403/5xx to `502` (keep 404), return a generic 5xx message outside development, and use one error shape (for example `application/problem+json`, also in `setNotFoundHandler`).

**21. Medium · Performance: zod response schemas add cost rather than speed**

- Problem: `fastify-type-provider-zod` runs `parse()` and then `JSON.stringify`, so it doesn't compile to fast-json-stringify. `MediaInfoSchema` is also parsed twice (once in the service at `:241`). Some routes have no response schema at all, and `@fastify/swagger` is installed but never registered.
- Fix: remove the duplicate parse, use `z.toJSONSchema()` on hot routes, and either register swagger or remove it.

**22. Medium · Reliability: outbound HTTP has gaps in timeouts and retries**

- Where: `tmdb.client.mts:52`. Stream, image, HLS, subtitle and ActiveEncodings fetches have no timeout, and `getSubtitleVtt` takes no abort signal.
- Problem: no retries for idempotent GETs, and `ApiClient` returns `{success:true} as T` for 204.
- Fix: `AbortSignal.any([clientSignal, AbortSignal.timeout(10_000)])`, one jittered retry on `ECONNRESET` for GETs, and `undefined` for 204.

**23. Medium · Ops: configuration is scattered and unvalidated, and NaN becomes a hot loop**

- Where: over 30 `process.env` reads.
- Problem: `BINKI_HEALTH_CHECK_INTERVAL_SECONDS=abc` leads to `setInterval(fn, NaN)`, which fires every **1 ms**. `${process.env.JELLYFIN_URL!}` becomes the string `"undefined"`.
- Fix:

**24. Medium · Reliability/Performance: SSE implementation**

- Where: `portalEvents.service.mts:8-40`.
- Problem: the return value of `res.write()` is ignored (no backpressure), there's no `retry:` or `id:`, the two endpoints are duplicates, and cleanup listens on `req.raw` `'close'`.
- Fix: send `retry: 5000`, use monotonic ids, drop slow clients, use `res.on('close')`, and delete `/api/portal/events`.

**25. Medium · Ops/Observability: logging**

- Problem: the custom string logger has no levels, no JSON and no request id, and Fastify's logger is disabled. Code mixes `console.*` and `logger`. Every image and HLS-segment request is logged at info level, and the webhook `?token=` is logged. `clientIp` trusts `X-Forwarded-For`.
- Fix: Fastify's pino (`logger: { level }`) with `request.log` child loggers carrying `correlationId`. Log image and HLS requests at debug, redact the token, and use `request.ip` and `reply.elapsedTime`.

**26. Low · Clean code: dead and leaky code**

- Unused: `tmdb.client.mts`, `getMediaByType`, `removeUserMediaFromList`/`purgePortalMediaAfterServerDelete`, `getItemsByIds`, `jellyseerrSubmissionExists`.
- `JOB_METRICS.durationMs` grows forever.
- `ApiStatsSerializer` calls `writeFileSync('./stats.json')` every 5 seconds, crashes on a corrupt file, and fails in a read-only container.
- `status/home.mts` is a 525-line HTML template literal.

**27. Low · Design patterns: service and client layering is inconsistent**

- `JellyfinService` and `JellyseerrService` are mostly pass-throughs, while other code calls the clients directly.
- Repos are free functions bound to module-singleton pools, while services are classes decorated on Fastify.
- Fix: pick one approach, either constructor injection in `servicesPlugin` or plain modules. Keep `ProcessorContext`.

**28. Low · Reliability: RabbitMQ and topology edge cases**

- `replaceJobsQueue` uses `deleteQueue(..., { ifEmpty: true })` on the main channel, so a concurrent publish can kill it; use `withScratchChannel`.
- `RetryScheduler` has no jitter.
- `startIngestWorker` reads env directly instead of `readIngestEnv()`.

**29. Low · Ops: Docker build and runtime**

- No `.dockerignore`, no `USER node`, and no server `HEALTHCHECK`.
- yt-dlp is unpinned and frozen at build time.
- The two Dockerfiles are about 90% identical.
- No BuildKit cache mounts, and `APP_VERSION` is unused.
- `env_file: .env` shares every secret with every container.
- Fix: add `.dockerignore` (`**/node_modules`, `**/dist`, `**/.env*`, `.git`), `USER node`, `init: true` and `healthcheck: test: ["CMD","node","-e","fetch('http://127.0.0.1:3113/health').then(r=>process.exit(r.ok?0:1))"]`. Pin yt-dlp, and use `RUN --mount=type=cache,target=/root/.yarn/berry/cache yarn install --immutable`.

**30. Low · Performance: minor hot-path items**

- `getUsers` makes sequential calls and does an O(n·m) `find`.
- `downloadBinaryToFile` buffers up to 20 MB in memory; stream it with `pipeline` instead.
- Reddit missing comments are fetched serially.
- `youtu.be` URLs aren't normalized, which defeats dedupe (`url.mts:25-31`).

### What's done well

- Optional-dependency boot with per-dependency backoff, cached health probes and feature gating. The note about not awaiting init in `onReady` shows real lifecycle awareness.
- Confirm channels plus `enqueued_at` plus the pending sweeper make Postgres the source of truth, so nothing is lost while RabbitMQ is down. Comment selections are merged in SQL.
- A TTL'd retry queue with a DLX back to jobs, retry counting from `x-death`, and careful queue-argument migration.
- Playback is streamed (`Readable.fromWeb`) with abort on disconnect, range validation and a traversal-safe HLS rewrite.
- Requests are claimed atomically with `INSERT … ON CONFLICT DO NOTHING RETURNING`, and `CATALOG_REMOVAL_FRACTION_LIMIT` protects against mass-unlinking.
- yt-dlp has a timeout, a stdout cap, the `--` guard, tree-kill and temp cleanup in `finally`.

### Top 5 actions

1. **Make shutdown real:** signal handlers, `init: true`, a grace period, consumer drain, and unref'd timers.
2. **Make ingest idempotent and redelivery-safe:** deterministic keys, a unique upsert, a quorum queue with a delivery limit, a job deadline, and multipart upload.
3. **Fix the catalog sync:** remove `hydrateAllMedia`, add `unnest` bulk upserts, and make it single-flight and throttled.
4. **Fix playback and request latency:** decide direct play vs. HLS once, and return 202 with the outcome over SSE.
5. **Add caching and limits for upstream calls, then clean up the contract:** a TTL LRU, `p-limit`, a paginated `/requests`, deletion of the legacy routes, coerced query types, one error envelope mapping upstream errors to 502, and a Zod config module.
