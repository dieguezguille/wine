# services.exe: forward the JIT pool and FEX environment to services

## Symptom

A 32-bit Windows service started through the service manager dies before its
first instruction. Steam's client service (`SteamService.exe /RunAsService`)
is the case that exposed it: Steam starts it for the CEG step of a protected
32-bit game, the service never answers, and Madeira Dock reports the step as
failed (`RemoteCallFailed`, EResult 55).

## Cause

`services.exe` builds every service's environment with
`CreateEnvironmentBlock(token, FALSE)`. That block is assembled from the
registry only, so none of the variables of the process that started the
session reach the service. Upstream Wine already works around this for
`WINEBOOTSTRAPMODE`.

On iOS the FEX modules depend on the environment:

- `WINE_IOS_JIT_RX`, `WINE_IOS_JIT_RW` and `WINE_IOS_JIT_SIZE` describe the
  JIT pool the app reserved. Without them the WoW64 FEX module of a 32-bit
  process cannot place its code cache and aborts.
- `FEX_*` carries FEX's whole configuration (TSO, host feature probe and so
  on). A service without it runs with a different configuration from every
  other process in the session, and `FEX_*` settings in `madeira.cfg` never
  reach services.

## Fix

After `CreateEnvironmentBlock`, walk the current environment of
`services.exe` and copy into the service block:

- `WINEBOOTSTRAPMODE` (the existing upstream behaviour, now part of the list),
- `WINE_IOS_JIT_RX`, `WINE_IOS_JIT_RW`, `WINE_IOS_JIT_SIZE`,
- every variable whose name starts with `FEX_`.

Nothing else is forwarded, so a service still gets the registry-derived
environment Windows gives it.

## Verification

On an iPhone (iOS 27.0.1) with Madeira 0.1.1, `services.exe` built from this
branch (aarch64-windows) lets Steam's 32-bit client service start, load
`SteamService.dll` and serve the client. The FEX log of the service process
shows the same `TSO config` line as the rest of the session, which confirms
that `FEX_*` reaches it.
