# sechost: an idle real-time ETW consumer instead of a failing stub

> Status: optional. This change was written while chasing the exit of Steam's
> 32-bit client service and was installed for every successful test, but the
> exit turned out to have another cause (see the madeira-dock branch that
> publishes `SteamPID` before the CEG step). It is not proven to be required.
> It is kept as its own branch so it can be reviewed, or dropped, on its own.

## Symptom

Steam's client service (`SteamService.exe`, 32-bit) opens a real-time ETW
consumer at start-up and runs `ProcessTrace` on a thread of its own. In
Madeira the log shows:

    fixme:eventlog:OpenTraceW ...: stub
    fixme:eventlog:ProcessTrace ... 1 00000000 00000000: stub

## Cause

`OpenTraceW` and `OpenTraceA` always fail with `ERROR_ACCESS_DENIED`, and
`ProcessTrace` returns `ERROR_CALL_NOT_IMPLEMENTED` at once. A program that
treats its consumer thread as long-lived sees that thread end immediately
after start-up. On Windows, a real-time consumer of a session that logs
nothing simply blocks in `ProcessTrace` until `CloseTrace`.

## Fix

- `OpenTraceW`: for a real-time consumer (`LoggerName` set, no
  `LogFileName`) return a fresh handle backed by a manual-reset event. Log
  file consumers keep failing as before. Only the leading fields of the
  structure are read, because Wine's `EVENT_TRACE_LOGFILEW` lacks
  `BuffersRead`, so its `LogFileMode` is not at the caller's offset.
- `ProcessTrace`: for handles that all belong to such consumers, wait until
  every one of them is closed and return `ERROR_SUCCESS`. Unknown handles keep
  returning `ERROR_CALL_NOT_IMPLEMENTED`.
- `CloseTrace`: signal and free the consumer; `ERROR_INVALID_HANDLE` for an
  unknown handle.
- `OpenTraceA` (advapi32): convert `LogFileName` and `LoggerName` and call
  `OpenTraceW` instead of failing on its own.
- `include/evntrace.h`: declare `OpenTraceA`/`OpenTraceW` and the `OpenTrace`
  macro, which the advapi32 change needs to build for i386.

No event is ever delivered: this only gives the consumer the lifetime it has
on Windows when the session is quiet.

## Verification

Built for i386 (`--enable-archs=i386`, the configuration of
`build/wine-i386/build.sh` in Madeira) and installed on an iPhone with
Madeira 0.1.1. The service's consumer thread stays in `ProcessTrace` for the
life of the service and the service is stopped cleanly by the service manager.
