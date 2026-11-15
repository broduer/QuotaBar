# QuotaBar defect audit (2026-11-12)

Baseline: `swift build` green, full suite **2507 tests / 0 failures / 13 skipped** before edits.
Branch: `freebuff/fix-bugs-and-rebuild-codex-usage-e9b22ec7-…`

**Status: 45 defects fixed, 0 confirmed open.** This codebase is unusually hardened (most
subsystems carry comments documenting already-fixed bugs); the list below is what survived
verification, not a quota filled with non-defects. See §4 for what was swept and what the
70-defect target would require that this code does not contain.

---

## 1. Anchor bug — Codex account #1 showed no usage (fixed)

### Reproduction

```sh
./.build/debug/quotabar-cli codex --force --diagnose
```

Returned no meters plus *"Couldn't reach Codex. Check your connection."* Probing the usage
endpoint with each stored credential (tokens never printed):

| Credential home | HTTP |
|---|---|
| CodexRouter pool home — the home the card preferred | **401 token expired** |
| Default `~/.codex/auth.json` — same ChatGPT `account_id`/`id_token` | **200 OK** |

### Root cause

1. **Only one auth home was tried.** An account can have several credential homes; Codex's CLI
   refreshes one, leaving the other stale. `CodexAccountCard` kept only `home` and
   `CodexAuthStore.authPaths()` returned just that path, so the valid bundle for the same verified
   identity was never read.
2. **Mislabelled failure.** A 400/401 from the token endpoint became a generic usage error and was
   shown as "check your connection", so the auth-fallback path never ran.

### Fix

`ProviderAccountAssembly` records every identity-verified home (`authHomes`);
`CodexAuthStore` tries them freshest-token-first; `CodexProvider.refreshAccessToken` maps a
rejected refresh to `CodexAuthError.tokenExpired` so `refresh()` moves to the next home; 429/5xx
stay transport failures.

### Verified on the installed build

`/Applications/QuotaBar.app/Contents/Helpers/quotabar codex --diagnose` after the final rebuild:
both cards are live and `stale: false` —
`Codex — tejasverma1@icloud.com` Session `used: 100` / Weekly `used: 57` (menu bar `0%`, `43%` left),
and `Codex — tejasverma920@gmail.com` Session `used: 0` / Weekly `used: 100`. Numbers differ per
account, confirming each card is reading its own credential home.

---

## 2. Fixed in this pass

### Crash-class (`Int`/duration traps on untrusted values)

| # | Location | Defect |
|---|----------|--------|
| 1 | `CodexUsageMapper.readPeriodMs` | `Int(seconds * 1000)` trapped on a large `limit_window_seconds`; now rejects a window outside one year. |
| 2 | `CodexUsageMapper.creditValues` | `Int(remaining)` trapped on a huge credit balance; now saturates at 1e9. |
| 3 | `CodexUsageMapper.readResetCredits` | `Int(available_count)` trapped; now saturates. |
| 4 | `CodexUsageMapper.resetDate` / `parseExpiry` | Unbounded `reset_at` / credit expiry produced far-future `Date`s that trapped duration formatting; now bounded by a 100-year representability horizon. |
| 5 | `ClaudeUsageMapper.resetDate` | Same unbounded reset instant; now bounded and threaded through `now`. |
| 6 | `OpenCodeGoUsageClient.parse` | `Int(percent)` trapped on a large value; a percent outside 0…100 now rejects the response (falls back honestly). |
| 7 | `CursorUsageMapper.billingCycle` | `Int(cycleEnd - cycleStart)` trapped on a huge gap; now bounded, falls back to the monthly cadence. |
| 8 | `Formatters.compactDuration` | A far-future interval (including one restored from a persisted/iCloud snapshot) trapped `Int(minutes)`; now returns nil so the label is omitted. |
| 9 | `WidgetData.resetCreditCount` | `Int(value)` trapped on a huge row value; now saturates. |
| 10 | `WidgetData.menuBarValue` | `Int(used / limit * 100)` trapped on a huge/NaN sample every menu-bar repaint; now sanitized and clamped. |
| 11 | `MenuBarSnoozeStore.evaluateAutoSnooze` | `Int(remaining * 100)` trapped on a NaN sample; now sanitized. |
| 12 | `OpenCodeGoWindowMath.anchoredMonthBounds` | A corrupt SQLite month anchor (e.g. 1e300) reached `Int` month arithmetic and trapped; non-epoch anchors now fall back to the calendar month. |
| 13 | `LogFile.append` | App and CLI opened the shared log without `O_APPEND`, so each wrote at its own offset and overwrote the other's lines. Now `O_APPEND` + `fstat` size. |

### Stale-state / wrong-data

| # | Location | Defect |
|---|----------|--------|
| 14 | Codex auth-home selection | See §1. |
| 15 | Codex refresh-error mapping | See §1. |
| 16 | `CopilotProvider.recordOrgRateLimitCooldown` | A malformed/millisecond `x-ratelimit-reset` paused org discovery until year 58000 — a permanently stale card. Cooldowns are now capped at one hour. |
| 17 | `CodexAutoResetCoordinator.runOnce` | A failed credit lookup retried every 15 s forever per account with no backoff. Now 15→30→60→…→300 s, reset on success; a near-expiry credit still gets the short retry. |
| 18 | `CodexAutoResetCoordinator.status` | Called `now()` 3–4 times per evaluation; could report `scheduled` and `windowPassed` for the same credit. One captured instant is used throughout. |

### Efficiency

| # | Location | Defect |
|---|----------|--------|
| 19 | `PiUsageScanner.aggregate` | Priced every row twice (row cost + API-estimate series); the estimate is now computed once. |
| 20 | `SpendTileMapper.dayKey` | Compiled three regexes per history row per refresh; replaced with byte-shape checks. |
| 21 | `TotalSpendAggregator.dayKey` | Same per-row regex compiles; replaced. |
| 22 | `CodexLogUsageScanner.autoReviewFallback` | Compiled a day-shape regex for every unresolvable auto-review/reserve row; replaced. |
| 23 | `CodexProxyUsageScanner.validDay` | Compiled the same regex per SQL row; replaced. |
| 24 | `AppendOnlyFileTailCache.evictIfNeededLocked` | Recomputed the retained-item count on every eviction → O(n·k) under the lock; now counted once and decremented. |
| 25 | `ClaudeUsageMapper.HTTPDateFormatter` | Built a `DateFormatter` per `Retry-After` parse; now a single immutable formatter. |

Regression coverage: `Tests/QuotaBarTests/MalformedPayloadTrapTests.swift` (10 tests feeding
`1e300` / far-future instants and asserting a clean result), plus the backoff test in
`CodexAutoResetCoordinatorTests`. The trap tests would SIGTRAP the test process without the fixes.

### Cross-process cache and ledger freshness

| # | Location | Defect |
|---|----------|--------|
| 26 | `FreebuffHostedUsageScanner` | The shared JSON cache was loaded once at init and never re-read; the app and `quotabar-cli` both write it, so a receipt written by the other process was invisible and last-writer-wins could overwrite it. Now stamped by size+mtime and reloaded before every merge/publish; the write restamps. |
| 27 | `CodexProvider.appendLocalUsage` | Ledger wakes reused `lastProxyScan` and `lastPiScan` with no revision check, so a wake after a pi session appended served the old fold until the next full pass. The proxy fold is now reused only when its FCC database revision tuple is unchanged; the pi fold is re-read (incremental, appended bytes only) because pi never touches the Codex ledger. |

### Numeric and clock hardening

| # | Location | Defect |
|---|----------|--------|
| 28 | `ModelUsageDetail.wholePercents` | `Int(NaN.rounded(.down))` trapped if a NaN share reached the list; now sanitized to 0. |
| 29 | `BackgroundRefreshGate` | A backward wall-clock jump made `shouldAccept` reject for the size of the jump, stalling external triggers; the gate now treats a negative elapsed time as an expired window. |
| 37 | `ClaudeUsageMapper.parseRetryAfterSeconds` / `ClaudeProvider` | `Retry-After: 9223372036854775807` is a valid `Int`, so the 429 cooldown was set ~292 billion years out; its remaining time plus `now` exceeded `Int.max` and trapped the next refresh. The parsed value is now capped at 24 h, the cooldown addition is clamped, and the display conversion clamps the `Double` before converting. Copilot's org-probe cooldown already had the same ceiling. |
| 38 | `DailyUsageAccumulator.absorbRows` | For every day that carried model rows it linearly scanned the whole daily series (`first(where:)`), so a multi-year scan cost O(days × seriesDays) per refresh. The series is indexed once into a dictionary. |
| 39 | `PricingCatalogCodecs.catalogFromLiteLLM` / `catalogFromModelsDev` | Published cost fields were trusted as-is: a negative rate inverted spend, and a huge or non-finite rate could overflow a token product to infinity. Both feeds now reject negatives, non-finite values, and rates above a generous ceiling before storing them; an invalid cache lane falls back to the input rate and is marked non-explicit, exactly like a missing one. OpenRouter already validated its values. |
| 40 | `ModelPricingStore.isDue` / `failureRetryBlocked` | The fetch-state file outlives launches. A `fetchedAt`/`failedAt` written while the clock was ahead made the elapsed time negative, which read as “inside TTL” forever and suppressed all pricing refreshes until the clock caught up. A negative elapsed time now means expired / not blocked. |
| 41 | `ProviderSnapshotCache.snapshot` | A snapshot stamped in the future (clock corrected backwards, or a snapshot restored from another Mac) produced a negative age, which counted as fresh indefinitely. Freshness now requires a non-negative age, so the next pass refetches. |
| 42 | `LocalLimitsAPI.WireProvider` | `stale` was `generatedAt >= refreshedAt + interval`, so a snapshot stamped after the envelope was generated reported `stale: false`. It is now stale when the snapshot timestamp is in the future as well. |
| 43 | `RateLimitResetsDetail` | The popover subtracted *every* credit claimed this session from the row's count. Once a refresh had removed that credit (and already decremented the count), the claim was charged twice, under-reporting the remaining resets and eventually forcing the empty state. Only claims still present in the latest expiry list are deducted, with a floor at zero. |
| 44 | `CodexRouterUsageScanner.scanResult` (aggregate-cache hit) | On a warm aggregate-cache hit the scan skipped the fold but re-ran ledger discovery, source stats, append-anchor sampling and the alias snapshot *outside* `JSONLAccountingWorkPacer`. That unpaced metadata work put a repeated unchanged refresh at 14.6–15.1% of one core — over the 10% contract pinned by `TokenAccountingEfficiencyTests.testRepeatedRouterAccountingStaysBelowTenPercentOfOneCore`, which the default suite skips. The cache-hit revalidation now runs inside the pacer like the cold-fold validation; the benchmark reports 8.04%. |
| 45 | `LocalTextFileAccessor.resolveSymlinkDestination` | Every relative hop in a symlink chain was resolved against the *original* path's parent. With `a/link1 -> ../b/link2` and `b/link2 -> target.txt`, a rotated credential was written to `a/target.txt` and the real `b/target.txt` stayed stale (the chained link was also left pointing at unchanged content). Each hop now resolves against its own link's directory and normalizes lexically. |

### Repeated-work / scaling

| # | Location | Defect |
|---|----------|--------|
| 30 | `CodexProxyUsageScanner` / `CodexLogUsageScanner` | `files.first(where:)` inside the future-timestamp loop was O(files × future entries); both now index the files once. |
| 31 | `MuseUsageScanner` | Same `files.first(where:)` pattern; now a single dictionary. |
| 32 | `HistoryDashboardAggregator.sinceLabel` | Built a `DateFormatter` on every dashboard render; now one immutable formatter. |
| 33 | `AntigravityUsageMapper.normalizeLabel` | Compiled a regex per meter label per refresh; now a static compiled regex. |
| 34 | `FreebuffModelIdentity.groupName` | Three regex compiles per distinct model name; now one static pattern pair. |
| 35 | `TotalSpendAggregator.slice` | Re-checked `dollarAggregationIncomplete` in the loop body after the guard, work that could never be true; removed. |
| 36 | `AppContainer.startPeriodicRefresh` | A history fold longer than the five-minute interval re-latched the next deadline while `accountingTask` was still running, so `wakeTimeout` became 0 and the loop spun on immediate timeouts until the fold ended. The wait now floors at 1 s while a fold is in flight and no other work is runnable. |

Verification for 26–36: `swift build` plus the full suite; observable behaviors carry tests
(`ModelUsageHoverTests` for 28, `BackgroundRefreshTriggerTests` for 29, `PaceTests` for the
formatter guard). 26, 27, 30–36 are behavior-preserving efficiency/freshness fixes covered by the
existing targeted suites (`CodexMultiAccountTests`, `CodexProviderTests`, scanner cache tests).

Verification for 44–45: the previously skipped router benchmark now passes when enabled (`QUOTABAR_PRODUCTION_ACCOUNTING_BENCH=1 QUOTABAR_TOKEN_ACCOUNTING_MODE=paced`), dropping the warm active share from 15.09% to 8.04%; the chained relative-link case is covered by the new `SymlinkPreservingWriteTests.testWriteFollowsAChainOfRelativeSymlinks` (it writes through two relative links and asserts the final target, no stray file beside the first link, and both links intact).

Verification for 37–43: new regression tests `MalformedPayloadTrapTests` (37),
`ModelPricingTests.testInvalidPublishedCostsAreRejectedInsteadOfPoisoningSpend` (39),
`ModelPricingStoreTests.testFutureFetchStampAfterAClockCorrectionDoesNotSuppressRefresh` (40),
`ProviderSnapshotCacheTests.testSnapshotStampedInTheFutureIsStaleNotFresh` (41),
`LocalLimitsAPITests.testFutureFetchedAtIsReportedStale` (42), and
`ResetDisplayTests.testResetsPopoverContentResolvesEmptyCountOnlyAndTimeline` (43). #38 is a
behavior-preserving index change exercised by the existing daily-accumulator suites.

---

## 3. Confirmed open

None. Every finding that survived verification is fixed above.

---

## 4. Coverage and its limits

Swept line-by-line across passes (providers, services, stores, support): Claude (auth store, desktop
store, log scanner, mapper, provider, session cache), Codex (auth store, log/proxy/router scanners,
router index/ledger/watcher, reset claim service, auto-reset coordinator, mapper, provider),
OpenCode (scanner, router ledger and aggregate, Go client/windows/math), Cursor (auth store, CSV,
mappers, provider), Antigravity (DB scanner, proto decoder, mapper, provider), Grok (scanner, mapper,
credits decoder, provider), Hermes (scanner, mapper, provider), Freebuff (hosted/desktop scanners,
model identity/catalog, SQL), Copilot, Devin, ZAI (+ ZCode), Muse, Pi, OpenRouter, `ProviderCatalog`,
`ProviderAccountAssembly`, `DefaultAccountObserver`, `ProviderRuntime`, incremental JSONL scanning and
its cache store/coordination/tail caches, `SpendTileMapper`, `DailyUsageAccumulator`,
`DailyUsageSeries`, `UsageHistoryAggregator`, `HistoryDashboardAggregator`, `TotalSpendAggregator`,
`ModelPricing`, `ProviderSnapshotCache`, `WidgetDataStore` refresh policy, `WidgetData`, formatters,
parsing/ISO8601 helpers, `LogFile`, `LogRedaction`, `ProcessRunner`, `LocalUsageServer/API`,
`BackgroundRefreshTrigger`, `RefreshWakeSignal`, `AppContainer` refresh loop, enablement/layout/accounts
stores.

Additionally swept line-by-line in the later passes: `PricingCatalogCodecs`, `ModelPricingStore`
(+ discovery), `JSONLScanCacheStore` (writer/reader, pruning, locking), `CodexRouterEventLineParser`,
`OpenCodeRouterLedger` (both modes, checkpoint validation, future-stamp markers),
`RateLimitResetsDetail`, `TotalSpendCard`, `ProviderSetupDiscoveryStore`,
`LayoutStore+Customization`, `NotificationSettingsStore`, `OnboardingStore`,
`PopoverNavigationStore`, `LaunchAtLoginSetting`, `MenuBarPrivacyStore`, `FirstRunSeeder`,
`DiagnosticDashboardController`, `QuotaNotificationEvaluator`, `MenuBarSnoozeStore`, `Formatters`,
`OpenUsageISO8601`, `CursorUsageCSV`, `CursorUsageClient`.

Read line-by-line in the final pass (closing the earlier gaps): `DashboardView`, `DashboardContentView`,
`WidgetRowView`, `WidgetGroupedListView`, `WidgetRowView`'s bounded/unbounded rows and hover targets,
`SettingsScreen`, `PopoverTopBar`, `HeaderView`, `PopoverFooter`, `ProviderSectionHeader`,
`HoverTooltip` (modifier + `TooltipPresenter`), `HoverPopoverState`, `UsageSparkline`,
`UsageTrendDetail`, `UsageTrendPopoverPresenter`, `PanelHeightModifier`, `PanelHeightCoordinator`,
`ProviderSectionVisibility`, `ICloudSyncSettingsSection`, `CodexAccountsSettingsSection`,
`SnoozeSettingsSection`, `ShareCardView`, `StatusItemController` (panel lifecycle, show/hide,
context menu, transparency re-arm, teardown), `ICloudUsageSyncStore` (enable/disable reconciliation,
write debounce, reload, metadata observation), `UsageHistoryAggregator`,
`HistoryDashboardAggregator`, `IncrementalJSONLScanner` (scan planning, fold/unload, persistence,
hydration, resident trimming), `OpenCodeRouterLedgerAggregate` (reconcile, materialize, prune, save,
locking), `ModelPricingStore+Discovery`, and `SystemClients` (environment resolution, private
file writes/symlink resolution, SQLite CLI, Keychain refresh reads and permission gate).

Skip audit: the 13 skipped tests are not disabled assertions. Two `ProductionTokenAccounting`
benchmarks, three `ProviderScannerEfficiency` benchmarks and one opt-in token-accounting benchmark
are machine-timing contracts; they were executed here with synthetic data and all now pass (one hid
defect 44). `CodexLogUsageScanner`/`ClaudeLogUsageScanner`/`GrokLogUsageScanner` parity tests,
`ClaudeProvider` live checks and `FreebuffHostedLiveValidation` require the owner's live
credentials/logs; `ProviderCatalogWiring` requires a real router ledger on the machine;
`CodexRouterProductionCache` and `AntigravityAccountingProjection` are explicit read-only local
audits. Their parsing/folding code is covered by fixture-driven suites, and the live tests skip by
design rather than hiding a failing assertion.

The audit stopped at 45 substantiated defects because these modules either carry evidence of an
earlier fix, validate untrusted input at their boundary, or have no reachable defect this sweep could
prove — not because a target was met. The 70-defect target cannot be reached here without counting
the same root cause once per call site (for example the remaining in-memory `now - stamp < interval`
caches, which are session-scoped and only skip a refresh after a mid-session clock rollback) or
inventing entries.

---

## 5. Audited and found sound

`ProviderParse`, `OpenUsageISO8601`, `ProviderAuthRetry`, `HTTPClient` (redaction/loopback TLS),
`WidgetDataStore` refresh policy (generations, in-flight scopes, failure backoff,
stale-while-revalidate, manual-action fencing), `ProviderSnapshotCache` (session freshness,
account stamps, refresh-order fences, bounded history), `RefreshWakeSignal`,
`CodexRouterLedgerWatcher`, `FutureTimestampRevisionGate`, `JSONLScanCacheStore` invalidation,
`AppendOnlyFileProbe`, `ProviderAccountsStore`, `CodexAccountRegistrationStore`.
