# Remote Client — Driving EPLAN from an External App

Namespaces: `Eplan.EplApi.RemoteClient`, `Eplan.EplApi.Remoting`, `Eplan.EplApi.Starter`.
Reference DLLs from `C:\Program Files\EPLAN\Platform\<version>\Bin\` (e.g. `Eplan.EplApi.RemoteClientu.dll`). For EPLAN 2025: target **.NET Framework 4.8.1**; transport is **gRPC** (add `Grpc.Core.Api`).

## Version gotchas (important)

| Version | Remoting behavior |
|---|---|
| 2023 | Remoting server on by default |
| 2025 | Must enable **"Remote Client Access"** in EPLAN options, otherwise no server/port (49152) opens. `EplanRemoteClient.StartEplan()` does NOT work (relies on the removed `EplanServer` action) — launch the process yourself and poll. |
| 2026 | Same as 2025. With the option off the process listens on **no** port at all, and a client gets gRPC `Unavailable: failed to connect to all addresses` — not a wrong-port problem. The option takes effect after an EPLAN restart. |

## Runtime and API assembly names per version

| Version | .NET runtime | Managed API assemblies | Detect it from `Bin` |
|---|---|---|---|
| 2026 and older | .NET Framework 4.8.1 (`ImageRuntimeVersion` v4.0.30319) | `Eplan.EplApi.Baseu.dll`, `...DataModelu.dll`, ... | no `Grpc.Net.Client.dll`, no `*Netu.dll` |
| 2027 | .NET 8 (coreclr) | `Eplan.EplApi.BaseNetu.dll`, `...DataModelNetu.dll`, ... — the `*u.dll` beside them are native twins | `Grpc.Net.Client.dll` and `*Netu.dll` present |

- A client or add-in must run on the same runtime: pythonnet loads `coreclr` for 2027 and
  the default .NET Framework runtime for 2026; a .NET 8 assembly loaded into EPLAN 2026
  fails with `TypeLoadException: Could not load type
  'System.Runtime.Loader.AssemblyLoadContext' from 'mscorlib, Version=4.0.0.0'`, and
  EPLAN survives it (measured live on 2026.0.3).
- Compiling is not proof: a `net8.0-windows` project builds cleanly against 2026's `*u.dll`
  (they are just metadata to the compiler) and still cannot load there.
- Choosing references: take every `Eplan.EplApi.*Netu.dll` when the folder has any, else
  `Eplan.EplApi.*u.dll` minus the `*Netu` ones. Never glob `*u.dll` on 2027 — it matches the
  native twins too.

## Which address the port binds

EPLAN 2026 binds its remoting port (49152 here) on the machine's **LAN addresses and `[::1]`,
not on `127.0.0.1`** (`netstat -ano` while connected). `Connect("localhost", ...)` still
works because `localhost` resolves to `::1`, but the port is reachable from the network:
treat anything that can run scripts over it (`ExecuteScript`) as exposed to the LAN.

## Minimal client (discover + connect + execute)

```csharp
using Eplan.EplApi.RemoteClient;

// 1. Discover running EPLAN instances (dynamic port!)
List<EplanServerData> servers = new List<EplanServerData>();
using (var probe = new EplanRemoteClient())
{
    probe.GetActiveEplanServersOnLocalMachine(out servers);
}
if (servers.Count == 0) { /* no EPLAN running */ }

// 2. Connect to the newest one
var target = servers.OrderBy(s => s.EplanVersion).Last();
var client = new EplanRemoteClient();
client.SynchronousMode = true;   // wait for actions to finish
client.Connect("localhost", target.ServerPort.ToString(), TimeSpan.FromSeconds(5));

// 3. Execute and clean up
client.ExecuteAction("ActionName");
client.Disconnect();
client.Dispose();
```

Never hardcode the port — always discover it via `GetActiveEplanServersOnLocalMachine`.

**From Python (pythonnet), the `out` list comes back as the call's return value** and the
list you passed in stays empty — so a port of the C# above that reads the argument always
sees "no servers" and falls back to a guessed port:

```python
from System.Collections.Generic import List
returned = client.GetActiveEplanServersOnLocalMachine(List[EplanServerData]())
# the out-list is the result, or the last item of a (return value, out...) tuple
servers = returned[-1] if isinstance(returned, tuple) else returned
```

Measured on 2027 and confirmed live on 2026.0.3 (it then reports `2026.0.3.25794` on port
49152). Enumeration can still come back empty for a while right after EPLAN starts, with the
port already up, so keep a fallback that reads the ports the EPLAN process listens on.

## Executing actions with parameters (`ActionContext`)

Remote actions use `Eplan.EplApi.Remoting.ActionContext` with `.Set(key, value)` (not `ActionCallingContext`):

```csharp
var context = new ActionContext();
context.Set("Template", templatePath);
context.Set("StorageLocation", outputDir);
context.Set("ProjectName", projectName);
context.Set("Overwrite", "True");
context.Set("BreakOnErrors", "False");
context.Set("ShowLog", "False");
client.ExecuteAction("CogineerGenerateFromExcelAction", ref context);
```

### Running a script inside EPLAN remotely
The workhorse pattern: heavy logic lives in a `.cs` script executed *inside* EPLAN; the external app only orchestrates:
```csharp
var context = new ActionContext();
context.Set("ScriptFile", @"\\server\Eplan\Scripts\MyExecutor.cs");
// extra Set() values are readable by the script
client.ExecuteAction("ExecuteScript", ref context);
```

### Receiving responses
```csharp
client.ResponseArrivedFromEplanServer += (EplanResponse response) =>
{
    // response.Succeed, response.Message, response.Parameters (dictionary)
};
// unsubscribe before Disconnect()
```

## Launching EPLAN headless and waiting for the server

`/Frame:0` = invisible main window. Poll for the remoting server (2025 needs Remote Client Access enabled):

```csharp
using Eplan.EplApi.Starter;

// Locate the installed EPLAN executable:
EplanFinder finder = new EplanFinder();
List<EplanData> versions = new List<EplanData>();
finder.GetInstalledEplanVersions(ref versions, true);
var eplan = versions
    .Where(v => v.EplanVariant.Equals("Electric P8") && v.EplanVersion.StartsWith("2025"))
    .OrderBy(v => v.EplanVersion).LastOrDefault();

// Launch headless:
var psi = new ProcessStartInfo
{
    FileName = eplan.EplanPath,
    Arguments = "/Variant:\"Electric P8\" /NoLoadWorkspace /NoSplash /Quiet /Frame:0"
};
Process eplanProcess = Process.Start(psi);

// Poll until the remoting server appears (up to ~90 s):
int waited = 0;
while (waited < 90)
{
    Thread.Sleep(3000); waited += 3;
    List<EplanServerData> found = new List<EplanServerData>();
    using (var probe = new EplanRemoteClient())
        probe.GetActiveEplanServersOnLocalMachine(out found);
    if (found.Count > 0) break;   // ready — connect now
}
```
EPLAN's process name is `W3u` (useful for `Process.GetProcessesByName` diagnostics).

## Connection resilience pattern

Production-proven approach (retry + ping + auto-reconnect):

- **Connect** with N retries and increasing backoff (`Thread.Sleep(2000 * attempt)`); on each attempt re-discover servers (the port may have changed).
- **Health check**: `client.Ping()` on a timer; on exception, mark disconnected and trigger reconnect in the background (`Task.Run(() => InitializeEplan())`), capped at MAX_RETRIES.
- **ForceReconnect** sequence: dispose current client → discover servers → if none, launch EPLAN headless and poll → connect to newest server.
- **Shutdown**: try `client.StopEplan()` first, wait ~3 s, then `process.Kill()` + `WaitForExit(5000)` as fallback; dispose everything.
- Log with area prefixes (`[CONNECT]`, `[PING]`, `[RECONNECT]`, `[DIAG]`) including exception type, message, and inner exception — remoting failures are opaque without this.

## Cogineer generation from Excel (TYPICAL workflow)

Full pipeline used to generate schematics from an Excel "TYPICAL":
1. Connect to EPLAN (start headless if needed).
2. `CogineerGenerateFromExcelAction` with `ActionContext` params: `DT`, macro params (`M...`), `Template` (.zw9), `StorageLocation`, `ProjectName`, `Overwrite`, `BreakOnErrors`, `SC`, `ShowLog`.
3. `ExecuteScript` with a documentation script (PDF export per language, labels, backup, final ZIP).
4. Externally watch the output folder (e.g. `DispatcherTimer` every 2 s) for the final artifact to detect completion — remote actions give limited completion signals for long scripts.
