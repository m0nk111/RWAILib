# RimBridgeServer Testing Guide

RimBridgeServer is a RimWorld 1.6 mod by Andreas Pardeike (the Harmony author) that
turns a running RimWorld session into a live automation bridge. We installed it
to help test and debug the RWAILib 1.6 port.

## Installation (Complete ✅)

- **Source repo**: `Q:\GAMES\RIMWORLD\modsworkspace\RimBridgeServer\`
- **Deployed to**: `Q:\GAMES\RIMWORLD\Rimworld.v1.6.4850\game\Mods\RimBridgeServer\`
- **Enabled in**: `C:\Users\onyou\AppData\LocalLow\Ludeon Studios\RimWorld by Ludeon Studios\Config\ModsConfig.xml`
- **Package ID**: `brrainz.rimbridgeserver`
- **Version**: 2.1.0
- **Pre-built DLLs**: Yes (9 DLLs in `1.6/Assemblies/`)

## How to Use

### Direct Mode (No GABS Required)

1. **Start RimWorld** — the bridge auto-starts
2. **Read the log** — look for these lines in `Player.log`:
   ```
   [RimBridge] GABP server running standalone on port 5174
   [RimBridge] Bridge token: abc123...
   ```
3. **Connect a client** to:
   - Address: `127.0.0.1`
   - Port: logged port (usually 5174)
   - Token: logged bridge token

### Via GABS (Recommended for AI/Automation)

1. Install [GABS](https://github.com/pardeike/GABS)
2. Configure a `rimworld` game entry
3. Start RimWorld through GABS
4. GABS exposes tools as `games.start`, `games.connect`, `games.call_tool`
5. The RimBridgeServer tool surface appears behind those

## Key Tools for RWAILib Port Testing

### Verify Mod Loading
```
rimworld/list_mods
```
Returns all installed mods, enabled status, and loaded session match.
Use this to verify RWAILib (AICore/AIItems) is loading correctly after port.

### Check Mod Configuration
```
rimworld/get_mod_configuration_status
```
Returns semantic mod-configuration status, warnings, ordering issues.

### Inspect UI Layout
```
rimworld/get_ui_layout
```
Returns structured layout of dialogs, windows, main tabs, inspect tabs, gizmos.
Use this to verify:
- AICore welcome dialog renders correctly
- Server status indicator appears in bottom-right
- Settings window works

### Take Screenshots
```
rimworld/take_screenshot?clipTargetId=<ui-element-id>
```
Capture full or cropped screenshots for visual verification.
Use this to verify:
- Welcome banner image displays
- Status icons render
- Art description replacement works

### Execute Debug Actions
```
rimworld/search_debug_actions?query=art
rimworld/execute_debug_action?path=<path>
```
Search and execute RimWorld's debug actions.

### Inspect Game State
```
rimworld/get_game_info
rimworld/list_colonists
rimworld/get_selection_semantics
```
Inspect live game state, colonists, selection details.

### Control Game Speed
```
rimworld/pause_game
rimworld/set_time_speed?speed=4
rimworld/step_game_ticks?ticks=60
rimworld/play_for?duration=5&speed=4
```
Control game timing for testing AI job processing.

### Run Lua Scripts
```
rimbridge/compile_lua?source=<lua-code>
rimbridge/run_lua?source=<lua-code>
```
Run automation scripts (lowered Lua subset).

### Check Bridge Status
```
rimbridge/get_bridge_status
rimbridge/list_logs
rimbridge/list_capabilities
```
Diagnose bridge health, read logs, discover available capabilities.

## Testing Workflow for RWAILib 1.6 Port

### Step 1: Verify Mod Loads
1. Start RimWorld with RimBridgeServer + AICore + AIItems enabled
2. Call `rimbridge/get_bridge_status` → verify bridge is healthy
3. Call `rimworld/list_mods` → verify AICore and AIItems are loaded
4. Call `rimbridge/list_logs` → check for errors

### Step 2: Test Welcome Dialog
1. Call `rimworld/get_ui_layout` → find welcome dialog
2. Call `rimworld/take_screenshot` → capture welcome dialog visually
3. Verify model size dropdown works
4. Verify Continue/Cancel buttons

### Step 3: Test Server Status Indicator
1. Call `rimworld/get_ui_layout` → find status indicator
2. Call `rimworld/take_screenshot?clipTargetId=<status-element>` → capture
3. Verify status shows correctly (Offline/Busy/Online/Error)

### Step 4: Test Art Description Replacement
1. Call `rimworld/start_debug_game_ready` → start test colony
2. Build or spawn a sculpture
3. Select it → call `rimworld/get_selection_semantics`
4. Call `rimworld/take_screenshot` → capture art description tab
5. Verify AI-generated text replaces default description

### Step 5: Test GlobalControls Patch
1. Call `rimworld/get_ui_layout` → inspect bottom-right controls
2. Call `rimworld/take_screenshot?clipTargetId=<controls-area>` → capture
3. Verify server status indicator appears in-game (not just main menu)

## Pre-built vs Build from Source

RimBridgeServer ships with pre-built DLLs in `1.6/Assemblies/`:
- `RimBridgeServer.dll` (main mod assembly)
- `RimBridgeServer.Core.dll` (core logic)
- `RimBridgeServer.Contracts.dll` (shared contracts)
- `RimBridgeServer.Sdk.dll` (SDK for companion tools)
- `RimBridgeServer.Extensions.Abstractions.dll`
- `Gabp.Runtime.dll` (GABP protocol runtime)
- `Lib.GAB.dll` (GAB library)
- `MoonSharp.Interpreter.dll` (Lua interpreter)
- `Newtonsoft.Json.dll` (JSON serialization)

To rebuild from source, install .NET SDK (6.0+), then:
```sh
dotnet build RimBridgeServer.sln -c Release
```

## Skills

RimBridgeServer includes two Codex/goose skills:
1. `rimbridge-server` — generated live-bridge skill (for running game interaction)
2. `rimbridge-companion-tools` — repo-owned skill (for adding companion DLL tools)

Install with: `scripts/install-skills.sh` (requires bash + python)
