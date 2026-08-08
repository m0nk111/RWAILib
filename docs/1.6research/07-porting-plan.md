# Porting Plan: RWAILib 1.5 → 1.6

## Executive Summary

Port RWAILib (AICore + AIItems) from RimWorld 1.5 to 1.6. The AIServer component
(Python) requires no changes. The main work is:
1. Create new .csproj files targeting 1.6 ref assemblies
2. Replace the fragile GlobalControls transpiler with a Postfix
3. Verify all Harmony patches work against 1.6 API
4. Update About.xml, solution, and build scripts
5. Build, test, and push to GitHub

---

## Phase 1: Project File Setup

### Step 1.1: Create AICore 1.6.csproj
- Copy `AICore/Source/1.5.csproj` → `AICore/Source/1.6.csproj`
- Change `Krafs.Rimworld.Ref` version: `1.5.*` → `1.6.*`
- Change `DefineConstants`: `RW15` → `RW16` (both Debug and Release)
- Keep same `ProjectGuid`

### Step 1.2: Create AIItems 1.6.csproj
- Copy `AIItems/Source/1.5.csproj` → `AIItems/Source/1.6.csproj`
- Change `Krafs.Rimworld.Ref` version: `1.5.*` → `1.6.*`
- Change `DefineConstants`: `RW15` → `RW16`
- Update `ProjectReference` to point to `../AICore/Source/1.6.csproj`
- Keep same `ProjectGuid`

### Step 1.3: Update Common.props (AICore)
- In `ZipMod` target: rename `Dir15` → `Dir16`, `Copy15` → `Copy16`
- Change ILRepack output path: `../../1.5/Assemblies/` → `../../1.6/Assemblies/`
- Update all `1.5` folder references to `1.6`

### Step 1.4: Update Common.props (AIItems)
- Same changes as AICore Common.props
- Update `Copy16` to not include Textures (AIItems doesn't have Textures at mod root)

### Step 1.5: Create 1.6 Output Directories
```
mkdir AICore/1.6/Assemblies/
mkdir AIItems/1.6/Assemblies/
```

### Step 1.6: Update Solution File
- Add new project entries for `1.6.csproj` files
- Keep existing 1.5 entries (or remove if going 1.6-only)

### Step 1.7: Update build.cake
- Change project paths from `1.5.csproj` to `1.6.csproj`

### Step 1.8: Update Directory.Build.props
- Bump `ModVersion` from `0.0.1.0` to `0.0.2.0`

---

## Phase 2: About.xml and Metadata Updates

### Step 2.1: AICore/About/About.xml
- Change `<li>1.5</li>` to `<li>1.6</li>` in `<supportedVersions>`
- Bump `<modVersion>` to `0.0.2.0`

### Step 2.2: AIItems/About/About.xml
- Change `<li>1.5</li>` to `<li>1.6</li>` in `<supportedVersions>`
- Bump `<modVersion>` to `0.0.2.0`

---

## Phase 3: Code Changes

### Step 3.1: Replace GlobalControls Transpiler (HIGH PRIORITY)

**Current**: Transpiler that injects IL into `GlobalControls.GlobalControlsOnGUI`
**Problem**: Extremely fragile, will likely break on 1.6

**Approach**: Replace transpiler with a Postfix.

The current transpiler inserts `PlayGUIServerStatusPatch(ref float num2)` into the
middle of `GlobalControlsOnGUI` to draw server status at the bottom-right, then
decrements the screen height counter.

A simpler, more robust approach:

```csharp
// Replace the entire transpiler class with a Postfix:
[HarmonyPatch(typeof(GlobalControls), nameof(GlobalControls.GlobalControlsOnGUI))]
public static class GlobalControls_GlobalControlsOnGUI_Patch
{
    public static void Postfix()
    {
        // Draw server status indicator at bottom-right corner
        // This runs after the original method completes
        // Position it at the bottom-right of the screen
        float sw = UI.screenWidth;
        float sh = UI.screenHeight;
        // ... (draw status box)
    }
}
```

**Note**: The original transpiler was carefully positioned to avoid overlapping
with play settings controls. The Postfix approach draws after all other controls,
which means the status box may be positioned differently. Test carefully and
adjust positioning as needed.

### Step 3.2: Verify CompArt API Compatibility
- Try building AIItems against 1.6 ref assemblies
- If `CompArt.titleInt`, `ITab_Art.cachedImageDescription`, `ITab_Art.cachedImageSource`
  fail to compile, check the 1.6 decompiled source for renamed fields
- Update field names if needed

### Step 3.3: Conditional Compilation (if needed)
If any APIs differ between 1.5 and 1.6, use conditional compilation:
```csharp
#if RW16
    // 1.6-specific code
#else
    // 1.5-specific code
#endif
```

---

## Phase 4: Build & Test

### Step 4.1: Restore NuGet Packages
```sh
cd RWAILib
dotnet restore RWAILib.sln
```

### Step 4.2: Build AICore
```sh
dotnet build AICore/Source/1.6.csproj -c Release
```

### Step 4.3: Build AIItems
```sh
dotnet build AIItems/Source/1.6.csproj -c Release
```

### Step 4.4: Verify Output
- Check `AICore/1.6/Assemblies/AICore.dll` exists
- Check `AIItems/1.6/Assemblies/AIItems.dll` exists
- Check `Libraries/` has compressed native libs

### Step 4.5: Install for Testing
Copy the mod to the game Mods folder:
```
Q:\GAMES\RIMWORLD\Rimworld.v1.6.4850\game\Mods\3269938006\
```
Or use RimPy to manage the mod installation.

### Step 4.6: Runtime Test
1. Launch RimWorld 1.6
2. Enable dev mode
3. Enable RimWorldAI Core and RimWorldAI Items mods
4. Check Player.log for errors
5. Test welcome dialog and server status
6. Test art description replacement

---

## Phase 5: GitHub

### Step 5.1: Create Branch
```sh
git checkout -b feature/1.6-port
```

### Step 5.2: Stage & Commit
```sh
git add .
git commit -m "Port to RimWorld 1.6: new .csproj files, About.xml updates, transpiler replacement"
```

### Step 5.3: Push to Fork
```sh
git push origin feature/1.6-port
```

### Step 5.4: Create Pull Request (optional)
Create PR on GitHub from `m0nk111/RWAILib:feature/1.6-port` to `m0nk111/RWAILib:main`.

---

## Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Transpiler breaks on 1.6 IL | 90% | High | Replace with Postfix (Step 3.1) |
| Private field renames (CompArt.titleInt etc.) | 30% | Medium | Build check, update if needed |
| MainMenuDrawer API changed | 20% | Medium | Verify, update if needed |
| gRPC native lib incompatibility | 5% | High | Verify lib loads at runtime |
| Python bootstrap fails | 5% | High | Test bootstrap flow |

## Estimated Effort

| Task | Effort |
|------|--------|
| Project file setup | 2 hours |
| About.xml updates | 15 min |
| Transpiler → Postfix rewrite | 2 hours |
| Compile error fixes | 1-3 hours (depends on API changes) |
| Testing | 2-4 hours |
| Documentation | 1 hour (already done) |
| **Total** | **8-12 hours** |

## File Change Summary

| File | Change Type | Description |
|------|------------|-------------|
| `AICore/Source/1.6.csproj` | NEW | New project file for 1.6 |
| `AIItems/Source/1.6.csproj` | NEW | New project file for 1.6 |
| `AICore/Source/Common.props` | MODIFY | Update ILRepack output, zip targets |
| `AIItems/Source/Common.props` | MODIFY | Update ILRepack output, zip targets |
| `AICore/Source/Patches.cs` | MODIFY | Replace GlobalControls transpiler with Postfix |
| `AICore/About/About.xml` | MODIFY | Update supportedVersions, modVersion |
| `AIItems/About/About.xml` | MODIFY | Update supportedVersions, modVersion |
| `AICore/Directory.Build.props` | MODIFY | Bump version |
| `AIItems/Directory.Build.props` | MODIFY | Bump version |
| `RWAILib.sln` | MODIFY | Add 1.6 project entries |
| `build.cake` | MODIFY | Update project paths |
| `AICore/1.6/Assemblies/` | NEW DIR | Build output directory |
| `AIItems/1.6/Assemblies/` | NEW DIR | Build output directory |
| `docs/1.6research/` | NEW DIR | Research documentation |
