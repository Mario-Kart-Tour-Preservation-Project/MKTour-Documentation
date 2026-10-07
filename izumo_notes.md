# Izumo client key: notes only (no implementation)

Static analysis of libil2cpp.so (C#) and libs2pcore.so (native Sakasho). Addresses are vaddrs.

## What the endpoint looks like

| | |
|---|---|
| Path / method | `POST /v3/booster/izumo/create_client_key` (libs2pcore export `SksIzumoCreateClientKey`, path + `HTTP_POST` referenced directly in the export) |
| C# -> native args | none besides callbacks (`SksIzumoCreateClientKey(IntPtr handle, SuccessCallbackWrapper, FailureCallbackWrapper, DebugOption)`) |
| Request body | The C# side builds no protobuf. `Sks.Protobuf.V3.Booster.Izumo.CreateClientKey.Request` exists in schema.dll and is **empty** (no fields), so an empty body is the natural reading, but the native request body was not decoded: UNKNOWN |
| Response | `Sks.Protobuf.V3.Booster.Izumo.CreateClientKey.Response { optional string value = 1; }` (from `Sks.Izumo.ResultOfCreateClientKey::.ctor(Response)`) |
| Key length limit | `nn.pia.izumo.IzumoNetworkSetting.cIzumoClientKeyLengthMax = 128` (`get_IzumoClientKeyLengthMax` @0x4f54f7c returns 0x80) |

What the key *is* (format, how the pvp server validates it, lifetime) is not visible in the client: UNKNOWN. The client only
stores the string and hands it to Pia.

## Flow through the client

```
Game.NetworkUtil.<routinePrepareToInitializeFramework>  (also routineCheckP2PAvailable, routineReInitializeFramework)
  -> Game.SakashoUtil.Izumo.CreateClientKeyAsync                 @0x3c928bc
  -> Game.SakashoIzumoManager.CreateClientKeyAsync               @0x3faebe8
  -> SakashoIzumoManager.routineCreateClientKey (coroutine d__5) @0x4f8307c
       -> Shelp.Izumo.Accessor.CreateClientKey                   @0x4af8e40
       -> Shelp.Internal.ApiCaller.dispatchIzumoApi
       -> Sks.Izumo.CreateClientKey                              @0x39378e0
            RegisterCallback<Izumo.ResultOfCreateClientKey, Response>
            closure <CreateClientKey>b__1 -> P/Invoke Sks.Izumo.SksIzumoCreateClientKey @0x460b2f8
       <- Sks.Izumo.<>c.<CreateClientKey>b__1_0 builds ResultOfCreateClientKey(Response) -> .Value = Response.value
  stored as SakashoIzumoManager.ClientKey (get_ClientKey @0x43202e0)

Game.NetworkManager.initializeFrameworkCore                      @0x43308a0
  -> Game.SakashoUtil.Izumo.get_ClientKey                        @0x37caa34
  -> nn.pia.izumo.IzumoNetworkSetting.set_clientKey              @0x3e2bc18
  -> P/Invoke SetClientKeyNative(IntPtr, string)                 @0x52c30a4
  -> native C export IzumoNetworkSetting_SetClientKey            @0x1f86644 (92 bytes) in libil2cpp.so (Pia C API)
```

The same `initializeFrameworkCore` also sets `WanNetworkSetting.SetApplicationId("mariokarttour")` and
`WanServiceLoginSetting.SetLoginServerAddress("pvp.mariokarttour.com,11401")`, so the key is used for the Izumo login to the
pvp server (see `hosts.md`). Whether the key is sent inside the 76-byte type-06 hello seen in the capture is still UNKNOWN.

## Error handling after the call

`routineCreateClientKey` checks the failure's `Shelp.ErrorCode` (client-side codes, `ldr w8,[x0]; cmp #imm`):

| Getter | Code |
|---|---|
| `IsClientVersionOld` | 1016 (0x3f8) |
| `IsPvPOverCapacity` | 1018 (0x3fa) |
| `IsUnderPvPMaintenance` | 2198 (0x896) |
| `IsPvPClientKeyInvalid` | 2200 (0x898) - also checked in `NetworkUtil.<routinePrepareToInitializeFramework>` to re-request a key |

These are Shelp's internal numbers, not the wire enum. The server-side error message type is
`Sks.Protobuf.Common.Error { ErrorCode code = 1; string message = 2; }`, whose enum includes `PVP_INVALID_CLIENT = 301`
and `PVP_OVER_CAPACITY = 302`. The mapping from wire codes to Shelp codes (`Sks.*.MapError`, Shelp error tables) was not traced:
UNKNOWN which wire code produces 2200.

## Open questions

1. Native request body for `create_client_key` (empty protobuf vs no body).
2. Key format/semantics and how the pvp (Izumo) server checks it.
3. Wire error code -> Shelp code mapping for the PvP errors.
4. Whether `IzumoNetworkSetting_GetClientKey` (@0x1f866a0) is used to echo the key anywhere else.
