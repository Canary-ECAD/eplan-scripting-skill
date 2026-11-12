# Critical Pitfalls

These are the failure modes that actually bite in production. Read before writing any multi-step EPLAN automation.

## 1. The command-blocking issue (message loop)

**Symptoms**: an EPLAN command (`reports`, `translate`, `backup`…) completes its work, but your C# code hangs after `oCLI.Execute()`; subsequent log lines never run; the process looks frozen while EPLAN itself is fine.

**Root cause**: EPLAN commands are **pseudo-asynchronous** — they run in EPLAN's context and need an active Windows message loop to signal completion. Without one, `Execute()` never returns.

**Solution**: keep a secondary thread pumping message monitoring while the main thread runs commands sequentially:

```csharp
bool continueMonitoring = true;
Thread monitorThread = new Thread(() =>
{
    while (continueMonitoring)
    {
        _messageMonitor.MonitorMessages();   // keeps the message loop alive
        Thread.Sleep(3500);
    }
});
monitorThread.Start();

try
{
    // Main thread: run EPLAN operations SEQUENTIALLY
    UpdateConnections(projectName);
    GenerateReport(projectName);
    Translate(projectName);
    Compress(projectName);
    BackupProject(projectName, exportDir, comment);
}
finally
{
    continueMonitoring = false;
    monitorThread.Join();       // never leave orphan threads
}
```

Non-solutions that were tried and failed: `Task.Wait()` with timeout, extra action parameters, sleeps/delays. The monitor thread is architectural, not a workaround.

## 2. Sequential execution model

All EPLAN operations against one instance are **strictly sequential** — each action must fully finish before the next starts (reports need synchronized data; backup needs compress done; etc.). Never parallelize actions against the same EPLAN instance. The only legitimate parallel thread is the message monitor above (it doesn't run actions).

## 3. `using` / `Dispose` discipline

`ActionCallingContext`, `EplanRemoteClient`, streams, forms — anything `IDisposable` — must be disposed even on exception. Missed disposals leak native resources and can wedge remoting connections.

```csharp
// Preferred:
using (ActionCallingContext acc = new ActionCallingContext())
{
    ...
} // disposed even if an exception is thrown

// Stacked:
using (var r1 = new Resource1())
using (var r2 = new Resource2())
{ ... }
```
For long-lived clients (e.g. a WPF app holding an `EplanRemoteClient`), implement the full `IDisposable` pattern: `Dispose()` → disconnect → stop process if you own it → dispose client → null it.

## 4. Error handling rules

**The commonest way to lose an error is to call an action the default way.**
`new CommandLineInterpreter()` does not transmit exceptions to the caller, so
a failed action gives you a bare `false`. Two things recover the cause:

```csharp
// ✅ Read the exception off the context - works on both executor paths,
//    no re-execution, no flags:
bool ok = new ActionManager().FindAction("projectmanagement").Execute(acc);
var ex = acc.GetException();      // the BaseException behind the false
// acc.SysMessages also carries this call's messages, with severity.

// ✅ Or ask the interpreter to transmit them (two other ctors exist):
new CommandLineInterpreter(true).Execute("someAction", acc);        // throws
new CommandLineInterpreter(true, true).Execute("someAction", acc);  // + acc.SysMessages
```

**Never re-run an action to harvest its message.** `false` does not mean
nothing happened — a `restore` returned false *after* completing an overwrite
that deleted unrelated files. Use the context you already have.

And know the limit: `GetException()` returns **null** for precondition
failures, which are silent on every channel. `SetProjectLanguage` with no
project open returns false with a null exception, empty `SysMessages`, and
nothing in the tree at any severity — and a *valid* language id fails
identically. When the exception is null, suspect a missing precondition, not
your parameters. (Measured on EPLAN 2027.0.1, 2026-09-03.)

```csharp
// ❌ NEVER
try { ... } catch { }

// ✅ Inside EPLAN scripts — surface into system messages:
catch (Exception ex)
{
    new BaseException("Error: " + ex.Message, MessageLevel.Error).FixMessage();
}

// ✅ In external apps — log everything remoting gives you:
catch (Exception ex)
{
    _logger.LogError($"[CONNECT] {ex.GetType().FullName}: {ex.Message}");
    if (ex.InnerException != null)
        _logger.LogError($"[CONNECT] Inner: {ex.InnerException.Message}");
}
```
- In batch loops (parts DB scans, page iterations): catch per item, log the item id, continue.
- In interactive scripts: `MessageBox.Show(ex.Message, "Error", ...)` is acceptable; never in headless paths.

**Reading the message tree back:** `BaseException` does *not* have a `.Level` property (CS1061 if you try it) — the real one is `.MessageLevel`, returning `Eplan.EplApi.Base.MessageLevel` (`Message`/`Warning`/`Error`/`FatalError`) per entry:

```csharp
var col = new SysMessagesCollection(0, MessageLevel.Message); // 0 = no bookmark filter
var it = col.GetSysMsgEnumerator();
while (it.MoveNext())
{
    var m = it.Current as BaseException;   // Current is object - cast first, CS1061 otherwise
    if (m != null)
        Console.WriteLine($"{m.MessageLevel}: {m.Message} (x{m.NumberOfOccurrences})");
}
```
`NumberOfOccurrences` collapses consecutive *identical* messages EPLAN joined into one tree item — don't rely on it as a reliable dedup count, it stayed `1` for messages fired back-to-back in a live test. Verified live 2026-09-01 against a running EPLAN 2025 instance.

## 5. Progress bars must always end

`Progress.EndPart(true)` in `finally`, or EPLAN's UI is left with a stuck progress dialog. Check `Canceled()` inside loops when `SetAllowCancel(true)`.

## 6. Long-running work completion detection

Long remote scripts give weak completion signals. Robust patterns:
- Script writes a sentinel file / final artifact (ZIP) → external app polls the folder (timer, 2 s).
- Script posts status to a local HTTP/SignalR endpoint (see integration-patterns.md).

## 7. File writing from scripts

- Overwrite checks before writing user-facing outputs (`File.Exists` → ask).
- Timestamped names for backups: `name_Backup_yyyy-MM-dd_hh-mm-ss`.
- Create directories before use: `Directory.CreateDirectory(dir)` is idempotent.
- EPLAN examples traditionally use `Encoding.Unicode` for text files it re-reads; use UTF-8 for anything consumed by other tools.

## 8. Miscellaneous

- Dialogs kill headless automation — suppress with `QuietModeStep(QuietModes.ShowNoDialogs)` around dialog-prone actions, and never `ShowDialog()` in automation paths.
- Action names and parameters are case-sensitive strings with zero compile-time checking — a typo fails silently or at runtime. Verify against the RAG.
- Ports are dynamic; process name is `W3u`; EPLAN 2025 needs "Remote Client Access" enabled (see remoting.md).

## 9. A "hung" or timed-out script may just be a swallowed compile error

If a generated script never returns / never produces its result and nothing
in the script logic looks slow, don't reach for RAM, project size, or a stuck
lock as the first theory. A script that fails to compile never runs at all,
so any host that waits for a result artifact (file, callback, whatever) just
sees a timeout — indistinguishable from a genuine hang until you check
EPLAN's own message tree (`SysMessagesCollection`, item 4 above) for a
`CS####`-prefixed entry naming the generated source file. See
`e3d-installation-spaces.md`'s Pitfalls section for a live-reproduced example
where this was mistaken for a RAM problem and then a project-size problem
across 5 attempts before the real cause (an invalid `using`) turned up in the
message tree.

### Which C# the engine accepts depends on the version
 on **2026** the script engine
compiles with a **pre-C# 6** compiler, verified by probe — `?.` gives
`CS1525`, and `new Dictionary<string, object> { ["a"] = 1 }` gives
`CS1525: Invalid expression term '['`. On **2027** a direct probe compiled
and ran that same dictionary index initializer. So the engine's C# level
moved somewhere between the two, and neither "it's C# 5" nor "modern C#
works" is safe to assume across versions.

Treat the table below as the **2026 (and earlier) floor**. Write to it when
a script has to run on a mixed fleet; probe first if you want to rely on
anything newer. A one-line probe settles it in seconds — a script that
compiles writes its result file, one that doesn't leaves a `CS####` in the
message tree.

These are all syntax errors on 2026, however normal they look:

| Feature | C# | Symptom | Write instead |
|---|---|---|---|
| `?.` `?[]` null-conditional | 6 | `CS1525: Invalid expression term '.'` + `CS1003: Syntax error, ':' expected` | explicit null check, or `Convert.ToString(x)` (yields `""` for null) |
| `$"text {x}"` interpolation | 6 | `CS1056` / parse errors | `string.Format(...)` or `+` |
| `new Dictionary<..> { ["k"] = v }` index initializer | 6 | parse errors at the `[` | construct, then `d["k"] = v;` |
| `nameof(x)` | 6 | `CS0103` | the literal string |
| expression-bodied members, auto-property initializers, `??=` | 6+ | parse errors | classic bodies |

**Why this is worse than a normal compile error:** the failure is invisible
to the caller. `ExecuteScript` still returns **success** in ~0.4 s, the
script never runs, and if your script's contract is "write a result file",
the only symptom is that the file never appears. Drive a script from an
external process with a wait-for-result loop and you get a *timeout* — so
you go debugging the connection, a modal dialog, or a "blocked" EPLAN,
while the actual `CS####` message sits in EPLAN's system-message tree.
Field-confirmed on EPLAN 2026: four separate parts-database functions were
written off as "EPLAN hangs on parts queries" for months; all four were
`?.` and one dictionary index initializer.

**So: on any script timeout, read the message tree first.**

```csharp
var col = new SysMessagesCollection(0, MessageLevel.Error);
var it = col.GetSysMsgEnumerator();
while (it.MoveNext())
{
    var m = it.Current as BaseException;   // .MessageLevel, not .Level (CS1061)
    if (m != null) Console.WriteLine(m.Message);
}
```

EPLAN brackets each failed compile with a `Compile errors ... in the script
<path>:` header and a `<path> cannot be compiled` footer, with the `CS####`
lines between them — and the generated file name is in both, so you can pick
out the messages belonging to one script. In this repo,
`_execute_script` in `scripted.py` does exactly that and reports
`compile_errors` instead of a bare timeout.

### A compile error is not the only silent timeout

Same symptom — no result file, caller reports a timeout — different cause.
Check these before assuming syntax:

- **A wrong member name is `CS1061`, i.e. still a compile error.** The trap
  is that the name looks right. `MDPart` has **no `ProductTopGroup`** member:
  that is the name of the enum *type*, and the property holding it is
  `GenericProductGroup`. Reflect over the type and read the real names rather
  than trusting a plausible one.
- **Leaving live EPLAN objects in what you serialize will hang the script.**
  An `MDPropertyValue` stored straight into the dictionary that
  `JsonConvert.SerializeObject` walks sends the serializer off through a
  native object graph, and the script never finishes. This one compiles
  fine — it is a genuine hang, not a compile error, so the message tree is
  empty. Flatten every value to a string/number *before* it goes in.
- **A runtime exception inside `[Start]`** — see the last note below.

Distinguishing them: if the message tree has a `CS####` for your script, it
is a compile error. If the tree is clean and the script still produced
nothing, suspect the serializer or an unhandled exception.

Two related notes:

- EPLAN pre-imports `System`, `System.Linq`, `Eplan.EplApi.Base`,
  `Eplan.EplApi.MasterData` and `Eplan.EplApi.Scripting`, so repeating those
  `using` directives logs `CS0105 (…appeared previously in this namespace)`.
  Harmless — the script still compiles — but it is noise in the tree, and it
  can distract from the one line that actually matters.
- A *runtime* exception inside `[Start]` is a different failure with the same
  outward symptom (no result file). Wrap the body in try/catch and write the
  exception into the result file, so the two cases stay distinguishable.

## 10. Don't `RegisterScript` a one-shot `[Start]` script

`RegisterScript` installs a script's *persistent* hooks (`[DeclareAction]`/
`[DeclareEventHandler]`/`[DeclareRegister]`, see script-basics.md). A
generated script that only has `[Start]` has none of those, so
`RegisterScript`ing it before `ExecuteScript` accomplishes nothing but an
EPLAN-side warning ("The script does not contain attributes for loading")
and two extra remote-API round-trips (register + unregister) per call —
measured at roughly 44% of total run time for a small script. Call
`ExecuteScript` alone for one-shot scripts; reserve `RegisterScript`/
`UnregisterScript` for scripts you're actually loading persistently.

## 11. Closing (and opening) one project without touching the user's

Automation that opens a scratch copy next to the user's real work must never
let the GUI decide which project an operation hits. Three things measured live
on EPLAN 2026.0.3 with a real project and two scratch clones open:

- **`XPrjActionProjectClose` takes no project argument and closes what the GUI
  has selected.** After a navigator selection had touched the clones, two close
  calls closed **three** projects, the user's real one among them. Checking the
  focused project before the call is not enough: a multi-selection is not
  visible through `selectionset /TYPE:PROJECT`.
- **Opening a project does not necessarily give it focus.** `ProjectOpen`
  reported success, the project was open, and `SelectionSet.GetCurrentProject`
  still returned the project the user was in. "Open it, then work on the current
  project" is therefore a write to the wrong project waiting to happen.
- **A project without focus is fully usable through its object.**
  `DMObjectsFinder` over it, property writes and page creation all worked on a
  clone that never had focus.

So resolve the project by path and act on the object:

```csharp
// C# 5 / reflection, as in references/e3d-installation-spaces.md
Type pmType = FindType("Eplan.EplApi.DataModel.ProjectManager");
object pm = Activator.CreateInstance(pmType);
object wanted = null;
List<string> before = new List<string>();
foreach (object p in (IEnumerable)pmType.GetProperty("OpenProjects").GetValue(pm, null))
{
    string link = (string)p.GetType().GetProperty("ProjectLinkFilePath").GetValue(p, null);
    before.Add(link);
    if (string.Equals(link, elkPath, StringComparison.OrdinalIgnoreCase)) wanted = p;
}
if (wanted == null) throw new Exception("Not open: " + elkPath);
wanted.GetType().GetMethod("Close", Type.EmptyTypes).Invoke(wanted, null);
// Re-read OpenProjects: exactly elkPath must be gone, nothing else.
```

`Project.Close()` closed only the clone, with the open-project list read before
and after as proof. Compare the lists every time and treat any other missing
project as a failure to report, not a detail.

## 12. "Does not match any existing software version" means "upgrade it first"

`ProjectOpen` on a project saved by an older EPLAN fails under QuietMode with
*"The project could not be opened because it does not match any existing
software version"* (Spanish UI: *"no se ajusta a ninguna versión de software
existente"*), and the wrapper may only report a generic open failure.
Interactively EPLAN would offer an upgrade; in batch nothing asks. Run
`XPrjActionUpgradeProjects /PROJECT:<path.elk>` first, then open — measured on
2026.0.3 with projects from 2023/2024. The upgrade is one-way for that copy and
writes a backup (`pr_V13.0_0.zw1` here) into the project's folder; clean it up
when that folder is a scratch area.

Copying `<name>.elk` + `<name>.edb\` under a new name is enough to clone a
project. The `.elk` is a UTF-16 LE text file of two lines (`<old>.edb` and a
format number such as `13.0`); its first line does not have to be rewritten — a
clone whose `.elk` still names the template's `.edb` opened fine after the
upgrade. If you do rewrite it, keep UTF-16 LE with BOM.
