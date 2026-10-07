# Boot sequence: BootScene -> main menu (Mario Kart Tour 4.0.0)

Reconstructed statically from the il2cpp dump: call graph (`scripts/build_xref_index.py`), method refs
(`scripts/il2cpp_refs.py`, `scripts/callpaths.py`), and the capture timeline in the preservation repo
(README section 7.4). This is **control-flow reachability and the init subtask list**, not a captured trace;
where ordering is inferred rather than proven it is marked. Addresses are libil2cpp.so vaddrs.

## 1. Process / engine start (no game network traffic)

1. `com.google.firebase.MessagingUnityPlayerActivity` (launcher) -> Unity player -> IL2CPP runtime; metadata is
   decrypted in `il2cpp_init` before any C# runs.
2. Scene `Assets/Main/Scene/BootScene.unity`. `Game.Boot.Awake`/`Start` -> `Game.Boot.BootStart` (coroutine
   `<BootStart>d__2`) -> `Game.Boot.StartMainScene` loads `BoosterMain.unity`.
3. `Game.SystemEngine.createManagers` constructs the manager singletons, including `Game.SakashoDirector..ctor`
   (0x4db46a8) and `NPF.NPFSDK` setup. No HTTP yet.

The first real network traffic in the capture (Bugsnag session, BaaS login, api.accounts, ~23x
api.mariokarttour.com, download-cdn, pubsub in t=0..60s) is driven by the Sakasho init below.

## 2. Title sequence triggers Sakasho init

`Game.UIInitializeSequence.<TitleSequence>d__0.MoveNext` (0x...; a large coroutine) runs the title UI and calls:

- `Game.UIInitializeSequence.InitializeSakasho` -> `Game.SakashoUtil.RequestInitialize` ->
  `Game.SakashoDirector.RequestInitialize` (0x4fb8b24) -> `routineInitialize` (coroutine `d__154`, body 0x4e651a0).
- `Game.UIInitializeSequence.WaitInitializeSakasho` (coroutine `d__19`) polls `SakashoUtil.get_InitState` until
  `Initialized`/`Error`, handling EULA, Nintendo-account-linked and maintenance dialogs.
- Later in the same sequence: `SakashoV2SecurityManager.RequestAppSetId` (0x3b79e68), which calls the Android
  plugin `jp.co.nintendo.booster.gsn.GsnUtil.sendAppSetIdRequest` (via `AndroidJavaClass`/`CallStatic`, decoded
  from the `<RequestAppSetId>d__2.MoveNext` string literals: `jp.co.nintendo.booster.gsn`, `GsnUtil`,
  `sendAppSetIdRequest`). App Set ID feeds Play Integrity.

## 3. SakashoDirector.routineInitialize (0x4e651a0) - ordered, from the decompiled coroutine

In program order of the coroutine body (so this ordering **is** from code, not inferred):

1. Wait `LocalSaveDataUtil.IsReady` (local save ready).
2. `createShelpConfig` (0x3f3f6f0): build `Shelp.Config` from `SakashoServerConfig.ServerURI` + `CommonKey`
   (see `hosts.md`) and a `TimeGetter` (`SakashoDirector.timeGetter` -> `Util.Time.get_TimestampUtc`, Unix seconds UTC).
3. `Shelp.Shelp.InitializeSystem(config, leaveBreadcrumbFunc)` (0x3bf7d80)
   -> `Shelp.SystemManager..ctor`/`Initialize` -> `Sks.System.InitializeSystem` (0x50cae4c)
   -> native `SksSystemInitializeSystem(serverURI, commonKey, market, acceptLanguage, userAgent-ish, ...)`.
   This also calls **`GET /v1/version`** (the native init helper issues the version request) and installs the
   TimeGetter. Device/app fields come from `NPF.NPFSDK.GetAppVersion/GetTargetedOS/GetRuntimeOSVersion/GetDeviceName/GetMarket`.
   Registers `SakashoUnshownErrorHandler` and `SakashoResponseDecodeErrorHandler`.
4. `Shelp.Shelp.Login` (0x423b230) -> `SystemManager.StartLogin` (0x47f2d54): sets HTTP proxy
   (`Sks.System.SetHttpProxy` from `GetProxyEnvironment`) and begins authorization.
   - Authorization path: `Sks.System.Authorize` closures -> **BaaS auth via NPF** (`NPF.NPFSDK.RetryBaaSAuth`,
     `NPF.User.BaaSUser.AuthorizationResult`) to obtain a BaaS id token, then
     `Sks.Internal.Session.CreatePlayerSession(idToken, count)` -> **`POST /v1/players/@me/session`**, whose
     `Response` carries `Sks.Protobuf.V1.Resources.Session { token, player_id, player_status }`. The session token
     becomes `X-Sks-Session-Token` on every later request (see `auth_flow.md`).
   - `SakashoDirector.onLoginFinished` records the result.
5. `dispatchSubTasks` (0x36893a8) + `checkAllSubTaskFinished`: fire the init subtasks (section 4).
6. `initializePostProcess` (0x3bf2f7c): the post-process subtasks (section 5).
7. Set `mInitState = Initialized`; `SystemCrashreportManager.NotifySakashoInitialized`.

## 4. Init subtasks (`SakashoDirector.ESubTask`, dispatched by `dispatchSubTasks` 0x36893a8)

The enum has 25 members; `dispatchSubTasks` kicks off the manager method for each (they run concurrently as async
tasks, so **order within this set is not guaranteed**). Manager method and the Shelp/Sakasho endpoint each one
reaches (static reachability; full data in `endpoints_map.csv`):

| # | ESubTask | Manager method | Endpoint(s) reached |
|---|---|---|---|
| 0 | Login | (done in routineInitialize step 4) | `POST /v1/players/@me/session` |
| 1 | GetMaster | `SakashoMasterManager.RefreshAll` | `GET /v1/masters` (Shelp.Master.Accessor.GetMastersAsync) |
| 2 | GetStorage | `SakashoStorageManager.RefreshAll` | `POST /v1/players/_/storages/list` (device account path) |
| 3 | GetNintendoAccount | `SakashoNintendoAccountManager.GetLinkedAccount` | `POST /v3/limited_nintendo_account/fetch_nintendo_account` |
| 4 | GetAssetBaseUrl | `SakashoAssetManager.GetBaseUrlAsync` | `GET /v1/assets/base_url` and/or `POST /v3/server_controlled_constant/get_all_constants` |
| 5 | GetPublicAnnouncements | `SakashoAnnouncementManager.InitAsync` | `GET /v1/players/@me/announcements` (via AnnouncementManager) |
| 6 | V2SecurityCheck | `SakashoV2SecurityManager.InitCheckAsync` | Play Integrity: `POST /v3/security_nonce/generate_nonce` then `POST /v3/daily_bonus/update_daily_bonus` (see auth_flow.md) |
| 7 | InitSubscriptionStatus | `SakashoSubscriptionManager.InitSubscriptionStatusAsync` | `/v3/limited_subscription/*` |
| 8 | CheckUnprocessedPurchase | `SakashoVirtualCurrencyManager.CheckUnprocessedPurchaseAsync` | `/v2/payment/received_extra_bonuses/list` (Shelp.Payment.Accessor) |
| 9 | InventoryNewFlag | `Game.Inventory.NewFlag.InitAsync` | `/v3/inventory/get_inventories` (inferred; not resolved statically) |
| 10 | FriendReqNewFlag | `Game.Friend.NewFlag.InitAsync` | `/v1/friend_requests` (inferred) |
| 11 | GreetCoinNewFlag | `SakashoGreetCoinManager.RefreshNewFlagAsync` | UNKNOWN (likely shared_resources) |
| 12 | SeasonSummaryNewFlag | `Game.SeasonSummary.NewFlag.InitAsync` | `POST /v3/booster/season_summary/list_summary_prepared_seasons` |
| 13 | InquiryNewFlag | `Game.Announcement.InquiryNewFlag.InitAsync` | inquiry status (NPF/Sakasho; Shelp.Inquery.Accessor) |
| 14 | GetEventCurrencies | `SakashoProductManager.RefreshEventShopAsync` | `GET /v1/booster/players/@me/event_race/terms` (Shelp.EventRace.Accessor) |
| 15 | RegistFirebaseToken | (not in the switch; handled elsewhere) | FCM token; `intentional_push`/analytics |
| 16 | SetupTreasureOnInit | `SakashoProductManager.SetupOnInitAsync` | products/lottery (UNKNOWN exact) |
| 17 | GetTreasureTicket | `SakashoProductManager.GetTicketsAsync` | `GET /v1/booster/players/@me/rarity_box_lottery/lotteries` (GetTickets) |
| 18 | GetLoginBouus [sic] | (not in the switch) | `/v2/players/@me/login_bonuses` |
| 19 | FetchMk8PlayReport | `SakashoMk8dxManager.Initialize` | `POST /v3/booster/mk8dx/fetch_play_report` |
| 20 | RecoverConsume | `SakashoVirtualCurrencyManager.RecoverOnlyConsume` | `POST /v1/players/@me/virtual_currencies/recover` |
| 21 | GetSpecialOfferPurchases | `SakashoProductManager.InitSpecialOfferAsync` | `/v2/payment/received_extra_bonuses/list` / products |
| 22 | InitializeMiiIcon | `SakashoPlayerMiiDataManager.InitializeIconCacheAsync` | `/v3/booster/mii/*` (list/search) |
| 23 | LoadWorldMii | `SakashoWorldMiiDataManager.LoadWorldMiiDataAsync` | `/v3/booster/mii/search_miis` (inferred) |
| 24 | SyncMiiData | `SakashoPlayerMiiDataManager.SyncMiiDataAsync` | `/v3/booster/mii/{create,delete,list_own}_miis` |

"inferred" / UNKNOWN rows are where static reachability did not land on a single endpoint (the manager method
reaches it through virtual `ApiHandleSet<T>` handlers that the static walker cannot always resolve). The endpoints
themselves are all in `endpoints_map.csv`.

## 5. Post-process subtasks (`SakashoDirector.ESubTaskPostProcess`, 8 members)

`InitWriteStorage`, `InitRankingManager` (`SakashoRankingManager.InitAsync`), `TreasureRecover`,
`InitFestirvalTeam`, `ReceiveOwnerDriver` (`SakashoEventRaceFestivalManager.UnlockOwnerDriverAsync`),
`JoinSuperWin` (`SakashoSuperWinManager.JoinSuperWinAsync`), `CheckQuickstartUnreceivedAchievement`
(`SakashoQuickStartManager.WaitUnreceivedAchievementIfNeed`), `RestoreUdemaeV301`
(`SakashoStorageManager.RestoreUdemaeHotfix`). These run after the main subtasks complete, before the menu opens.

## 6. Into the menu

Back in `<TitleSequence>d__0.MoveNext`, after `WaitInitializeSakasho` reports `Initialized`:
load menu UI resources, `SendSystemInitTimeLog`/`SendUserOwnResourceLog`/`SendUserSettingLog` analytics,
`UIEventManager.NotifyMenuInEnd`, main menu opens. Multiplayer (Pia/Izumo) is **not** part of boot; it starts only
when the player enters online play (`NetworkManager.initializeFrameworkCore`, see `izumo_notes.md`), matching the
capture (first pvp traffic at t~736s).

## Minimum a server must answer to reach the menu

In rough order: `GET /v1/version` -> BaaS login (NPF, out of Sakasho scope) -> `POST /v1/players/@me/session`
-> then the concurrent batch, of which the client blocks on at least: `/v1/masters`,
`/v1/players/_/storages/list`, `/v1/assets/base_url` (+ `get_all_constants`), announcements, and the
Play Integrity pair (`generate_nonce` + `update_daily_bonus`) unless security is stubbed. The rest can return
empty/default bodies to still reach the menu (needs testing).
