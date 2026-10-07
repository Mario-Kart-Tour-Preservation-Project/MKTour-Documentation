# MKTour Documentation: protocol and boot/auth write-ups

Human-readable documentation of the Mario Kart Tour 4.0.0 client/server protocol, recovered by static
analysis of a runtime il2cpp dump (no servers were contacted; all from local copies of the game files).
The machine-usable schemas and the endpoint map live in the Server contribution (`protos/`,
`data/endpoints_map.csv`).

| File | Contents |
|---|---|
| `hosts.md` | Server hosts and `SakashoServerConfig` (API host, web-view host, CommonKey, environment 99); Pia/Izumo pvp + NAT-check hosts; what comes from the server vs. the binary. |
| `auth_flow.md` | The two auth systems and the exact sequence a server must satisfy: Nintendo BaaS signed-JWT login -> session token -> `X-Sks-*` headers; the Play Integrity flow and the `SksV2SecurityVerifyPlayIntegrityJWT -> /v3/daily_bonus/update_daily_bonus` finding. |
| `boot_sequence.md` | BootScene to main menu: `SakashoDirector.routineInitialize` order, the 25 init subtasks and the endpoints each reaches, and the minimum a server must answer to reach the menu. |
| `izumo_notes.md` | Notes on `create_client_key` and how the key flows into Pia (`IzumoNetworkSetting_SetClientKey`) for the pvp login. Notes only; open questions listed. |

Addresses in these docs are `libil2cpp.so` virtual addresses (dump rebased to base 0). Findings are marked
with confidence and `UNKNOWN` where the evidence did not resolve.
