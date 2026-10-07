# Hosts and server configuration (Mario Kart Tour 4.0.0)

Sources: `work/il2cppdumper/stringliteral.json`, `dump.cs`, static xrefs from `scripts/build_xref_index.py`,
disassembly via `scripts/disasm.py` / `scripts/il2cpp_refs.py`. Addresses are libil2cpp.so vaddrs (base 0).
Nothing here was obtained by contacting any server.

## 1. Game API (Sakasho)

| Item | Value | Evidence |
|---|---|---|
| API base URL | `https://api.mariokarttour.com` | string literal 0x5b8b270, only user `Game.SakashoServerConfig$$setupSettings` @0x53c63d4 |
| Web view base | `support.mariokarttour.com` (used as `https://{0}`) | literal 0x5b26f38 in setupSettings; `SakashoWebViewContentManager$$GetBaseUrl` @0x4a263f8 formats `"https://{0}"` with `SakashoServerConfig.get_WebViewBaseURI` |
| CommonKey | 32 hex characters, string literal at 0x6b318d0 (value not repeated here; read it from `stringliteral.json`) | setupSettings stores it in `ServerConnectionSetting.CommonKey` |

### SakashoServerConfig.setupSettings (0x53c63d4), decoded from disassembly

```
sConnectionSettings = new Dictionary<SakashoDefine.EEnvironment, ServerConnectionSetting>();
var s = new ServerConnectionSetting();          // SystemBase subclass
s.ServerURI      = "https://api.mariokarttour.com";   // field 0x10   (str  x9,  [x20,#0x10])
s.WebViewBaseURI = "support.mariokarttour.com";       // field 0x18   (stp  x10, x8, [x20,#0x18])
s.CommonKey      = <literal 0x6b318d0>;               // field 0x20
sConnectionSettings.Add((EEnvironment)0x63 /* 99 = Product */, s);
```

Only **one** environment is registered in 4.0.0: `Product` (99). The enum still lists `Sandbox_401` ...
`Sandbox_439` (values 0-38) and `GetEnvironmentString` (0x3e276b8) maps them to `"401"`..`"439"` / `"product"`,
but no sandbox connection settings exist in this build.

### How the environment is chosen

- `SakashoServerConfig.load` (0x4a893d8): `UnityWebRequest.Get(string.Format("{0}/{1}", Application.streamingAssetsPath, "SakashoServerConfig"))`.
- `<routineLoadSetting>d__22.MoveNext` (0x414c158): on success, `JsonUtility.FromJson<SakashoServerConfig>(downloadHandler.text)`.
  Fields: `mEnvironment` (0x20, `SakashoDefine.EEnvironment`), `mUseAssetMaster` (0x24, bool).
  The APK's asset is `{"mEnvironment":99,"mUseAssetMaster":true}` (preservation repo README section 5).
- `get_CurrentConnectionSetting` (0x46ee978): `ContainsKey(mEnvironment) ? dict[mEnvironment] : <other branch>`.
  The other branch was not decoded (UNKNOWN what happens for an unregistered environment).
- `SakashoDirector.createShelpConfig` (0x3f3f6f0) copies `ServerURI` and `CommonKey` into `Shelp.Config`, which is
  passed to `Shelp.Shelp.InitializeSystem` and on to the native `SksSystemInitializeSystem(serverURI, commonKey, ...)`
  (see `endpoints_map.csv`, row for `/v1/version`). What libs2pcore does with `commonKey` is **UNKNOWN** (not analysed yet).

### ResourceConfig (`mServer = 8`)

The TextAsset `ResourceConfig` (`{"mServer":8,"mRevision":4396301,"mSakashoEnvironment":99,"mCourseSet":"default","mUserName":""}`)
has **no consumer in the 4.0.0 metadata**: no type has fields `mServer` / `mSakashoEnvironment` / `mCourseSet`, and there is no
`"ResourceConfig"` string literal. Meaning of `mServer = 8`: **UNKNOWN**; it looks like a leftover asset.

## 2. Multiplayer (Pia / Izumo)

| Item | Value | Evidence |
|---|---|---|
| Izumo login server | `pvp.mariokarttour.com,11401` (host,port in one string) | literal 0x5750db8; `NetworkManager$$initializeFrameworkCore` @0x43308a0 passes it to `nn.pia.wan.WanServiceLoginSetting.SetLoginServerAddress`; `NetworkDefine$$GetIzumoServerAddress` @0x4819e94 returns the same literal |
| Pia WAN application id | `mariokarttour` | literal 0x5a5afb0 -> `nn.pia.wan.WanNetworkSetting.SetApplicationId` in initializeFrameworkCore |
| NAT check primary | `nat-check-0.mariokarttour.com:34543` | `NetworkDefine..cctor` @0x3994ba4: `string.Format("nat-check-0.mariokarttour.com:{0}", cNATCheckPort)` -> `cIzmNATCheckPrimaryAddress` (static 0x50) |
| NAT check secondary | `nat-check-1.mariokarttour.com:34543` | same, -> `cIzmNATCheckSecondaryAddress` (static 0x58); `cNATCheckPort = "34543"` (static 0x48) |
| Room/relay servers | not in the binary | delivered by the pvp server at runtime (capture shows `ec2-...:14106`) |
| NPLN host template | `%s.lp1.p.srv.nintendo.net` | libil2cpp native Pia strings (preservation repo), not a C# literal |

The capture also showed NAT traffic to UDP 33334; that port is not a C# literal (it is Pia-internal or derived) - UNKNOWN.

## 3. Assets / CDN

- No `akamaized` / `cloudfront` host is present in any string literal.
- The asset base URL comes from the server: init subtask `GetAssetBaseUrl` -> `SakashoAssetManager.GetBaseUrlAsync`
  -> `GET /v1/assets/base_url` -> `Sks.Protobuf.V1.Assets.BaseUrl.Response { optional string base_url = 1; }`.
- Announcement images etc. similarly come from server data (CDN hosts seen only in the capture).

## 4. Nintendo platform hosts

- NPF / BaaS (`a6913b7b80402974409b47079c29d775.baas.nintendo.com`) is **not** a C# literal; the Java NPF SDK reads it from
  `assets/npf.json` (preservation repo). The C# side (`NPFSDK.dll`) only wraps the Java SDK.
- Web links shown in the in-game browser (`Game.SakashoUtil.WebViewContent`):
  `https://accounts.nintendo.com` (0x5830d40), `https://accounts.nintendo.com/withdraw/confirm` (0x6c53e58),
  `https://www.nintendo.co.jp` (0x575bba0), `https://mariokarttour.com/{0}/ranking/allcup` (0x595bd20),
  and a development URL `http://devepd-booster-guide-002.boy.nintendo.co.jp/tour_summary/sample.html` (0x5b13b78, used by
  `GetTourSummaryUrl` @0x3aa6be4; whether it is reachable in release builds is UNKNOWN).

## 5. Third party (sinkhole candidates)

`https://notify.bugsnag.com`, `https://sessions.bugsnag.com`, `https://notify.insighthub.smartbear.com`,
`https://sessions.insighthub.smartbear.com`, `https://www.gstatic.com/firebase/ssl/roots.pem`,
`https://api.twitter.com/...` (Sns.Twitter), Firebase/Play endpoints from the Java side.

## Summary for a private server

A redirected client needs, at minimum: the Sakasho API host (`api.mariokarttour.com`, HTTPS, protobuf), the NPF/BaaS host
(JSON, Java SDK), and for multiplayer `pvp.mariokarttour.com:11401` plus the two NAT check hosts on port 34543.
`support.mariokarttour.com` and the CDN host from `/v1/assets/base_url` are needed for web views and downloadable assets.
