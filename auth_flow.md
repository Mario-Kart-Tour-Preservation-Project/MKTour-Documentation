# Auth and security flow: what a server must satisfy (Mario Kart Tour 4.0.0)

Static analysis only. Evidence: Sakasho native request builders decompiled with rz-ghidra
(`work/s2pcore_*_pdg.txt`), C# call graph (`scripts/*`), generated protos (`out/protos/`), and the NPF/Java
findings from the preservation repo README section 6.3. Addresses are vaddrs (libil2cpp.so, or libs2pcore.so where
noted). No server was contacted.

There are **two independent auth systems**: Nintendo BaaS (NPF, JSON, Java) and Sakasho (protobuf, native). The
BaaS id token is the bridge: BaaS issues it, Sakasho exchanges it for a session token.

---

## 1. NPF / BaaS login (JSON, Java `NPFSDK`)

This runs first, inside `Shelp.Shelp.Login` -> `Sks.System.Authorize`, which calls `NPF.NPFSDK.RetryBaaSAuth`
(C# wrapper over the Java SDK). The C# side cannot see the request body; the Java side (preservation repo) does:

- `POST https://<baasHost>/core/v1/gateway/sdk/login` where
  `<baasHost> = a6913b7b80402974409b47079c29d775.baas.nintendo.com` (from `assets/npf.json`).
- Body includes a signed **client assertion JWT**:
  - `key   = "com.nintendo.zaka:" + <signing-cert SHA-1>`
  - `secret = TOTP(HMAC-SHA1, key, step = 600 s, digits = 8)`
  - `assertion = JWT{alg: HS256}.{ iss: key, iat: now, aud: "https://<baasHost>" }` signed with `secret`
- Response: a BaaS **access token** (Bearer, used for other NPF calls) and a BaaS **id token** (the JWT handed to
  Sakasho in step 2).

**Server implications**
- The assertion is a lightweight app-authenticity check, fully reproducible from public values (package name +
  signing-cert SHA-1 + wall clock). A private BaaS can verify it or accept any assertion.
- **Re-signing the APK changes the signing-cert SHA-1**, so both the TOTP key here and Play Integrity (section 4)
  change. A private BaaS must accept the new key or skip the check. This is the single biggest client-side
  coupling for a rebuild.
- Nintendo Account (`accounts.nintendo.com`, OAuth2+PKCE, client_id `5dbf84c0c704b31b`) is only involved when the
  player links an NA; the device-account path does not need it.

A minimal stub: accept the login, return a signed-looking BaaS id token string (any value Sakasho will forward),
plus an access token. Sakasho does not validate the BaaS token's signature itself - the Sakasho **server** does.

---

## 2. Sakasho session creation (protobuf, native)

`Sks.Internal.Session.CreatePlayerSession(idToken, count)` -> native export
`SksInternalSessionCreatePlayerSession(handle, onSuccess, onFailure, idToken, count)` (libs2pcore 0x54c...), which
builds the request in `fcn.00562450` and posts it.

- **`POST /v1/players/@me/session`**, `Content-Type: application/x-protobuf`.
- Request type: the generated contract is `Sks.Protobuf.V1.Sessions.Create.Request { optional string id_token = 1;
  optional string device_identifier = 2; }`. (endpoints_map marks the request UNKNOWN because this one is assembled
  in a native helper, not a C# closure; the contract above is the matching protobuf-net type and the native builder
  takes `idToken` as an argument - treat as high-confidence request, medium-confidence that field 2 is always sent.)
- Response type: `Sks.Protobuf.V1.Sessions.Create.Response { optional Sks.Protobuf.V1.Resources.Session session = 1; }`
  with `Session { optional string token = 1; optional string player_id = 2; optional PlayerStatus player_status = 3; }`
  and `PlayerStatus { EXISTING_PLAYER = 0; NEW_PLAYER = 1; }`.
- `session.token` is stored and used as `X-Sks-Session-Token` on all later Sakasho requests.

**Server must:** validate the forwarded BaaS id token (this is where BaaS and Sakasho meet), create/lookup the
player, and return a `Session` with a token. For a private server, "validate" can be "trust any id token".

---

## 3. X-Sks-* request headers (every Sakasho request)

Built natively in libs2pcore (decompiled; the header strings are assembled inline as little-endian immediates in
`work/s2pcore_headers_pdg.txt`). The request-envelope functions and what they set:

| Function (libs2pcore) | Sets |
|---|---|
| `fcn.0056b570` | the base header set (calls the ones below): adds `X-Sks-Current-Time`, `X-Sks-Verbose`, then session/title/protocol/request-id, and conditionally the method-override group |
| `fcn.0056b8ec` | `X-Sks-Session-Token` = session token (from the stored `Session.token`) |
| `fcn.0056b81c` | `X-Sks-Request-Id` and `X-Sks-Req-Id` (a per-request id/uuid) |
| `fcn.0056b9b4` | `X-Sks-Protocol-Version` |
| `fcn.0056c138` | `X-Sks-Title-Id`, `X-Sks-Req-Id`, `X-Sks-Protocol-Version`, `Content-Type: application/x-protobuf`, `User-Agent`, `X-Sks-Current-Time`, `X-Sks-Verbose` on the POST/GET request object, and the POST path/version line |
| `fcn.0056c8cc` | `Content-Type`, `Accept`, `X-Sks-Device-Timezone`, `X-Sks-Market`, plus extra per-request headers from a list |
| `fcn.0056dc28` | reads the response's `X-Sks-Time-Lag` header (parsed as a base-10 integer) to compute clock skew |

Observed header set (also in libs2pcore strings / preservation README 6.2): `X-Sks-Session-Token`,
`X-Sks-Title-Id`, `X-Sks-Protocol-Version`, `X-Sks-Market`, `X-Sks-Accept-Language`, `X-Sks-Device-Timezone`,
`X-Sks-Current-Time`, `X-Sks-Time-Lag`, `X-Sks-Request-Id`, `X-Sks-Req-Id`, `X-Sks-Verbose`, `User-Agent`,
`Content-Type: application/x-protobuf`.

- `X-Sks-Current-Time`: Unix **seconds** UTC. The C# TimeGetter (`SakashoDirector.timeGetter` ->
  `Util.Time.get_TimestampUtc` @0x39ccdd8) computes `(int)(DateTime.UtcNow - epoch).TotalSeconds`. Confirmed by
  disassembly (DateTime.UtcNow - stored epoch -> TimeSpan.TotalSeconds -> fcvtzs to int32).
- `X-Sks-Time-Lag`: server-provided; the client reads it from the response to track skew (`fcn.0056dc28`), and sends
  its own value back on later requests. A server can return `0`.
- **No per-request body signing (e.g. HMAC over the body) was found.** libs2pcore contains only a generic POCO
  `HMACEngine` template and the Poco `HTTPDigestCredentials` machinery (unused for these calls). The decompiled
  request builders add the headers above and the raw protobuf body, with no computed signature over the body. So a
  server does not need to verify a body MAC. (Marked: absence of evidence; a signature computed far from the header
  code could still exist, but none was seen in the builders or on the hot path.)

**Server must:** accept the headers, require a valid `X-Sks-Session-Token` (reject with a
`Sks.Protobuf.Common.Error { code, message }` - see the ErrorCode enum, e.g. `INVALID_SESSION = 38`), and may
ignore `X-Sks-Current-Time`/`Time-Lag`.

---

## 4. Play Integrity check, and why it posts to `/v3/daily_bonus/update_daily_bonus`

Driven by init subtask `V2SecurityCheck` -> `Game.SakashoV2SecurityManager.InitCheckAsync` (0x4dc2878). The two
native exports and their **actual** paths (from `scripts/analyze_s2pcore.py`, confirmed by decompiling each export
and reading the inline path string):

| C# API | native export | path string in the export | wire |
|---|---|---|---|
| `Sks.V2Security.GenerateNonce` | `SksV2SecurityGenerateNonce` (libs2pcore 0x5617fc) | `/v3/security_nonce/generate_nonce` (0x345aeb) | `Request { security_nonce_fields=10 }` -> `Response { SecurityNonce { nonce=1 } }` |
| `Sks.V2Security.VerifyPlayIntegrityJWT` | `SksV2SecurityVerifyPlayIntegrityJWT` (libs2pcore 0x561b30) | **`/v3/daily_bonus/update_daily_bonus`** (0x391ff9) | `Request { jwt=1 }` -> empty Response |

### This is not a mislabel - the export really posts to the daily-bonus path

`SksV2SecurityVerifyPlayIntegrityJWT` (0x561b30) was decompiled: it calls the shared request builder
`fcn.0056bdf8(req, &path, body, HTTP_POST)` with `path = cstr(0x391ff9) = "/v3/daily_bonus/update_daily_bonus"`.
Compare `SksV2SecurityGenerateNonce` (0x5617fc), byte-for-byte the same builder but with
`path = cstr(0x345aeb) = "/v3/security_nonce/generate_nonce"`. So the VerifyPlayIntegrityJWT export genuinely sends
the Play-Integrity JWT to the **daily_bonus** endpoint. The matching contract
`Sks.Protobuf.V3.DailyBonus.UpdateDailyBonus.Request { optional string jwt = 1; }` confirms it: the daily-bonus
update request's only field is a `jwt`, and the C# `Sks.V2Security.VerifyPlayIntegrityJWT(callback, responseToken)`
passes exactly one string (`"jwt"`, the Play Integrity response token). The C# response callback is
`RegisterCallback<ResultOfVerifyPlayIntegrityJWT>` with **no** schema type (`NONE_PARSED`), i.e. the client ignores
the response body.

**Most likely reading:** on this backend, the server-side Play Integrity verification is folded into the daily-bonus
update handler - submitting the integrity JWT and advancing the daily bonus are the same server call. The separate
`SksV2SecurityVerifyPlayIntegrityJWT` symbol is the Sakasho-SDK-generic name; Booster wired it to the daily_bonus
path. (Confidence: high that the export posts there; medium on the "folded into daily bonus" interpretation, since
the server handler is not in these binaries.) There is also a `VerifySafetyNetJWS` C# method (legacy SafetyNet)
sharing the same `ApiHandleSet<ResultOfVerifyPlayIntegrityJWT, NoValue>` handler; only the Play Integrity path is on
the modern boot flow.

### Play Integrity sequence a server must satisfy

1. Client: `POST /v3/security_nonce/generate_nonce` (`Request.security_nonce_fields`, usually `"*"`).
   Server returns `SecurityNonce { nonce }`.
2. Client: requests a Play Integrity token from Google Play, using that nonce and the GCP cloud project number
   `SakashoV2SecurityManager.cGCPProjectNumber = 166790037073` (static `Nullable<long>` in `.cctor` 0x50ea364,
   built as `0x26d575fe51`; same as the Firebase sender id). App Set ID (`RequestAppSetId`) also feeds this.
3. Client: `POST /v3/daily_bonus/update_daily_bonus` with `Request { jwt = <Play Integrity response token> }`.
   Server verifies the token (nonce match, package name, signing-cert digest, cloud project number) and returns an
   (empty) success; the client discards the body.
4. Errors surface as `Shelp.ErrorCode` values 2297-2317 (`V2Security_PlayIntegrity_*`, e.g. `NonceTooShort`,
   `CloudProjectNumberIsInvalid`, `AppNotInstalled`); the `EErrorCodeShelp` mirror is 8400-8417.

**Server implications:** Play Integrity cannot be reproduced by a private server because it needs Google to sign the
token for *this* app identity, and re-signing the APK breaks that anyway. A private server should **accept any jwt**
on `/v3/daily_bonus/update_daily_bonus` (and return a nonce on `generate_nonce`). This is the second major
client-side coupling after the BaaS TOTP key.

---

## 5. Minimum auth a private server must implement

1. **BaaS stub** (`baas.nintendo.com` host, redirected): accept `POST /core/v1/gateway/sdk/login` with any
   assertion; return an access token + an id token.
2. **`POST /v1/players/@me/session`**: accept the forwarded id token; return
   `Sessions.Create.Response { Session { token, player_id, player_status } }`.
3. Honour `X-Sks-Session-Token` on subsequent requests; no body-MAC verification needed.
4. **`POST /v3/security_nonce/generate_nonce`** -> return a nonce; **`POST /v3/daily_bonus/update_daily_bonus`** ->
   accept any `jwt`, return empty success. (Equivalently, make the client skip security - but that needs a client
   patch, whereas accepting-any works unmodified.)
5. `GET /v1/version` early in init (returns a version/ok).

## 6. Open questions (auth-specific)

- Exact `Sessions.Create.Request` fields actually sent natively (id_token confirmed; device_identifier likely).
- Whether `commonKey` (the 32-hex value in `SakashoServerConfig`) is used in any request signing inside libs2pcore,
  or only as an app/client identifier. The request builders on the hot path did not use it; **UNKNOWN**.
- The server-side semantics of `update_daily_bonus` beyond integrity verification (does it also grant the daily
  bonus and return it elsewhere?). The response is not parsed by the client.
- (resolved) `cGCPProjectNumber = 166790037073`.
