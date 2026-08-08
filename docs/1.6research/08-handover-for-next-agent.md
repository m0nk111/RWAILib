# Handover Document: Next Agent

## Project Status

**As of**: August 2026
**Current Phase**: Research complete, ready to start implementation
**Repository**: `Q:\GAMES\RIMWORLD\modsworkspace\RWAILib`
**Fork**: [github.com/m0nk111/RWAILib](https://github.com/m0nk111/RWAILib)
**Branch**: `main` (create `feature/1.6-port` before making changes)

## What Has Been Done

1. ✅ Cloned RWAILib from GitHub fork (m0nk111/RWAILib)
2. ✅ Fixed submodule URLs (pointed to igoforth originals) and initialized all 3 submodules
3. ✅ Analyzed complete codebase: AICore (C#), AIItems (C#), AIServer (Python)
4. ✅ Researched RimWorld 1.6 modding documentation from multiple sources
5. ✅ Created comprehensive research docs in `docs/1.6research/`
6. ✅ Identified all Harmony patches and their 1.6 risk levels
7. ✅ Created detailed porting plan with step-by-step instructions

## What Needs To Be Done

Follow the porting plan in [07-porting-plan.md](07-porting-plan.md), but in summary:

### Immediate Next Steps (Phase 1-3)
1. Create `1.6.csproj` files for AICore and AIItems (copy from 1.5.csproj, change ref version)
2. Update `Common.props` files to point to `1.6/` output directories
3. Update `About.xml` files: change `1.5` → `1.6`, bump version
4. **Replace the GlobalControls transpiler** with a Postfix (this is the highest-risk change)
5. Update `RWAILib.sln` and `build.cake` for new project files
6. Create `1.6/Assemblies/` output directories

### Build & Test (Phase 4)
7. Run `dotnet restore` and `dotnet build`
8. Fix any compile errors (likely CompArt private field renames)
9. Install built mod to game folder and test
10. Verify all Harmony patches work (especially the modified GlobalControls)
11. Test the full AI pipeline (bootstrap → server → art replacement)

### Publish (Phase 5)
12. Commit and push to `feature/1.6-port` branch
13. Create PR if desired

## Key Decisions Already Made

### Decision 1: Single-version (1.6 only) approach
Instead of using LoadFolders for multi-version support, we're going 1.6-only.
The fork is specifically for 1.6 porting. Original 1.5 support remains in the
original author's repo.

### Decision 2: Replace transpiler with Postfix
The `GlobalControls_GlobalControlsOnGUI_Patch` transpiler is the highest-risk item.
It searches for specific IL patterns that will almost certainly change in 1.6.
Replacing it with a Postfix eliminates this fragility entirely. See document 05
for the specific replacement approach.

### Decision 3: Krafs.Rimworld.Ref 1.6.* (wildcard)
Using `Version="1.6.*"` instead of pinning to a specific version. This allows
NuGet to pick the latest 1.6.x ref assembly (currently 1.6.4871).

## Critical Files to Modify

| File | Priority | Description |
|------|----------|-------------|
| `AICore/Source/Patches.cs` | 🔴 CRITICAL | Replace transpiler (lines ~280-400 approx) |
| `AICore/Source/1.6.csproj` | 🔴 CRITICAL | New file, copy from 1.5.csproj |
| `AIItems/Source/1.6.csproj` | 🔴 CRITICAL | New file, copy from 1.5.csproj |
| `AICore/Source/Common.props` | 🟡 HIGH | Update ILRepack paths |
| `AIItems/Source/Common.props` | 🟡 HIGH | Update ILRepack paths |
| `AICore/About/About.xml` | 🟡 HIGH | Update supportedVersions |
| `AIItems/About/About.xml` | 🟡 HIGH | Update supportedVersions |
| `RWAILib.sln` | 🟢 MEDIUM | Add 1.6 project entries |
| `build.cake` | 🟢 MEDIUM | Update project paths |

## Code Pattern: Transpiler Replacement

The current transpiler (`GlobalControls_GlobalControlsOnGUI_Patch`) needs to be
replaced. Here's the key code to change:

**REMOVE** (current transpiler with pattern matching):
```csharp
public static IEnumerable<CodeInstruction> Transpiler(IEnumerable<CodeInstruction> instructions)
{
    // ... complex IL pattern matching ...
}
private static void PlayGUIServerStatusPatch(ref float num2) { ... }
```

**ADD** (simple Postfix):
```csharp
public static void Postfix()
{
    // Draw server status at bottom-right of screen
    // Similar to what PlayGUIServerStatusPatch did
    // But called after GlobalControlsOnGUI completes
}
```

The status drawing logic from `PlayGUIServerStatusPatch` can be largely reused,
just without the `ref float num2` screen height manipulation.

## Reference Locations

| Item | Path |
|------|------|
| Game installation | `Q:\GAMES\RIMWORLD\Rimworld.v1.6.4850\game\` |
| Assembly-CSharp.dll | `Q:\GAMES\RIMWORLD\Rimworld.v1.6.4850\game\RimWorldWin64_Data\Managed\Assembly-CSharp.dll` |
| Installed mod (reference) | `Q:\GAMES\RIMWORLD\Rimworld.v1.6.4850\game\Mods\3269938006\` |
| RimPy mod manager | `C:\PROGRAMME\RimPy_Windows\RimPy.exe` |
| Build workspace | `Q:\GAMES\RIMWORLD\modsworkspace\RWAILib\` |
| AICore source | `Q:\GAMES\RIMWORLD\modsworkspace\RWAILib\AICore\Source\` |
| AIItems source | `Q:\GAMES\RIMWORLD\modsworkspace\RWAILib\AIItems\Source\` |
| AIServer source | `Q:\GAMES\RIMWORLD\modsworkspace\RWAILib\AIServer\` |
| Research docs | `Q:\GAMES\RIMWORLD\modsworkspace\RWAILib\docs\1.6research\` |

## External Resources

- **Krafs.Rimworld.Ref 1.6**: [NuGet](https://www.nuget.org/packages/Krafs.Rimworld.Ref/) version 1.6.4871
- **RimWorld Wiki 1.6 Mod Updates**: [Wiki](https://rimworldwiki.com/wiki/Modding_Tutorials/RimWorld_1.6_Mod_Updates)
- **Official 1.6 Modder Primer**: [Google Doc](https://docs.google.com/document/d/e/2PACX-1vRKE9u5ZW_zG45pxzwNvy4sxvozDeqtxlxpac5jwenOeW6liQCPgmPl9bIbtcMuqL1NPIDHOLFg64M_/pub)
- **Official 1.6 Changelog**: [Google Doc](https://docs.google.com/document/d/e/2PACX-1vRCjqVtPQDFGu4POiKTUd_8o3U2Asdhx99SOvcgU66ABdYtk3Cgndd53yJ6BC4tZX530pp_m6lf4Z9P/pub)
- **Decompiled 1.6 code**: [GitHub: Dyyrlysh/RimworldDecompile](https://github.com/Dyyrlysh/RimworldDecompile)
- **RimWorld Discord**: `#mod-development` channel

## Key Technical Notes

1. **AIServer (Python) needs NO changes** — it's game-version agnostic
2. **The mod uses ILRepack** to merge all dependencies into one DLL per mod
3. **The mod uses Krafs.Publicizer** to access private game fields — if a private
   field is renamed in 1.6, the build will fail (good — fail-fast, not silent)
4. **gRPC native libraries** are shipped as .gz compressed files and decompressed
   at runtime — no version dependency
5. **The bootstrap.py** downloads llamafile, Python, and Phi-3 model at first run —
   this is version-agnostic
6. **27 language translations** are already present and don't need updating

## Git Notes

- The main repo (`RWAILib`) is the fork at `m0nk111/RWAILib`
- Submodules point to `igoforth` originals (fixed during research phase)
- Submodule URLs were fixed via `git config submodule.*.url` but `.gitmodules` was
  NOT updated — consider fixing `.gitmodules` too for consistency:
  ```
  [submodule "AICore"]
      url = https://github.com/igoforth/RWAI_Core.git
  [submodule "AIItems"]
      url = https://github.com/igoforth/RWAI_Items.git
  [submodule "AIServer"]
      url = https://github.com/igoforth/RWAI_Server.git
  ```

## Estimated Remaining Work

- Implementation (Phase 1-3): ~4-5 hours
- Build & Test (Phase 4): ~2-4 hours (depends on compile errors)
- Git push (Phase 5): ~30 min
- **Total remaining**: ~6-10 hours
