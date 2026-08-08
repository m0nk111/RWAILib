# Debugging and Testing for RimWorld 1.6

## Development Mode

### Enabling Dev Mode
1. Start RimWorld
2. Go to **Options** in the Main Menu
3. Under **Gameplay**, check the **"Development Mode"** box
4. Dev mode remains on until explicitly turned off (persists across restarts)

### Dev Mode Tools
- **Debug logging menu** (Output Tab): Information, data, and charts
- **Incidents**: Spawn/trigger game events
- **Spawn weapon/apparel/pawn**: Test items
- **Toggle job logging**: Diagnose pawn AI
- **Draw pawn debug**: Visual pawn AI debugging
- **Lords**: View group AI controllers

## Unity / Mono Runtime (1.6)

As of RimWorld 1.6 (Odyssey):
- **Unity**: 2022.3.35
  - [Release notes](https://unity.com/releases/editor/whats-new/2022.3.35f1)
- **Mono**: unity-2022.3-mbe fork
  - [GitHub](https://github.com/Unity-Technologies/mono/tree/unity-2022.3-mbe)
- **.NET target**: net472 (still Mono-compatible, unchanged from 1.5)

## Debugging Setup

### Rider (Recommended)
Gareth's RimWorld plugin for Rider:
- [Plugin page](https://plugins.jetbrains.com/plugin/21583-rimworld-development-environment)
- Provides seamless debugging, build integration, and RimWorld-specific tools

### Doorstop (Manual Debugging)
For non-Rider setups or if the plugin isn't used:
- [pardeike/Rimworld-Doorstop](https://github.com/pardeike/Rimworld-Doorstop)
- Hooks into Unity's Mono runtime to enable .NET debugger attachment

### Visual Studio
1. Build mod in Debug configuration (produces .pdb files)
2. Launch RimWorld with debugger attached
3. Use breakpoints in C# code

### Log Output
RimWorld writes logs to:
```
Windows: %USERPROFILE%\AppData\LocalLow\Ludeon Studios\RimWorld by Ludeon Studios\Player.log
Linux:   ~/.config/unity3d/Ludeon Studios/RimWorld by Ludeon Studios/Player.log
macOS:   ~/Library/Logs/Unity/Player.log
```

RWAILib's LogTool also buffers logs and flushes them during `LongEventsOnGUI`.

## Testing Checklist for 1.6 Port

### Pre-Build Tests
- [ ] .NET SDK installed (6.0+ required for SDK-style projects)
- [ ] `Krafs.Rimworld.Ref` 1.6.* restores correctly
- [ ] All NuGet packages restore without errors
- [ ] `Krafs.Publicizer` generates publicized assemblies for 1.6

### Build Tests
- [ ] AICore compiles with `1.6.csproj` in Debug mode
- [ ] AICore compiles with `1.6.csproj` in Release mode
- [ ] AIItems compiles with `1.6.csproj` in Debug mode
- [ ] AIItems compiles with `1.6.csproj` in Release mode
- [ ] ILRepack runs successfully (output DLL in `1.6/Assemblies/`)
- [ ] Native libraries compressed to `Libraries/`
- [ ] No compile errors related to API changes

### Runtime Tests
- [ ] Mod loads without errors in RimWorld 1.6
- [ ] No red errors in log on startup
- [ ] Welcome dialog displays on first run (main menu)
- [ ] Model size dropdown works
- [ ] "Continue" button starts bootstrap process
- [ ] Bootstrap downloads curl → llamafile → AIServer.pyz → model
- [ ] Server status indicator shows "Busy" during bootstrap
- [ ] Server status indicator shows "Online" after bootstrap
- [ ] Server status indicator shows "Offline" when disabled
- [ ] Settings window opens and saves correctly

### AIItems Feature Tests
- [ ] Visit a sculpture → job is submitted
- [ ] Server processes job → returns description
- [ ] Art description is replaced with AI text
- [ ] Title is replaced with AI title
- [ ] Cached results persist across viewings
- [ ] Multiple art pieces handled correctly

### Harmony Patch Tests
- [ ] `LongEventHandler.LongEventsOnGUI` Postfix runs (check log output)
- [ ] `MainMenuDrawer.MainMenuOnGUI` Postfix runs (UI elements visible)
- [ ] `GlobalControls.GlobalControlsOnGUI` — **transpiler may fail** → check log for error
  - If replaced with Postfix: verify status indicator appears in-game
- [ ] `Current.Notify_LoadedSceneChanged` Postfix runs (coroutines start)
- [ ] `UIRoot.UIRootUpdate` Postfix runs (periodic tasks execute)
- [ ] `CompArt.GenerateImageDescription` Postfix runs (art replaced)

## Common Issues & Solutions

### "Cannot find GenUI.DrawTextWinterShadow in GlobalControls.GlobalControlsOnGUI!"
**Cause**: Transpiler IL patterns don't match 1.6 code.
**Solution**: Replace transpiler with Postfix (see patch audit document).

### "Type or namespace 'CompArt' could not be found"
**Cause**: Krafs.Rimworld.Ref not restoring or wrong version.
**Solution**: Verify `Version="1.6.*"` in .csproj, run `dotnet restore`.

### "AICore.dll could not be loaded"
**Cause**: ILRepack output missing or references mismatched.
**Solution**: Clean build, ensure ILRepack runs to completion.

### gRPC native library loading fails
**Cause**: Native lib not decompressed or wrong architecture.
**Solution**: Verify `Libraries/` folder has compressed libs; check `BootstrapTool.SetGrpcOverrideLocation()`.

### Server process won't start
**Cause**: Python runtime or bootstrap.py not downloaded.
**Solution**: Check `%USERPROFILE%\AppData\LocalLow\Ludeon Studios\RimWorld by Ludeon Studios\RWAI\` directory.

## Useful Tools

| Tool | Purpose | Link |
|------|---------|------|
| ILSpy / dnSpy | Decompile Assembly-CSharp.dll | [GitHub](https://github.com/icsharpcode/ILSpy) |
| RimWorld Doorstop | Runtime debugging | [GitHub](https://github.com/pardeike/Rimworld-Doorstop) |
| RimWorld Developer Plugin for Rider | IDE integration | [JetBrains](https://plugins.jetbrains.com/plugin/21583-rimworld-development-environment) |
| TDBug | Dev/debug enhancements | [GitHub](https://github.com/alextd/RimWorld-TDBug) |
| RimPy | Mod manager | Installed at `C:\PROGRAMME\RimPy_Windows` |
