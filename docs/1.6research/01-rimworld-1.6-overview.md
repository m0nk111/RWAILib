# RimWorld 1.6 Overview

## Release Information

- **Version**: 1.6.4850 (latest patch as of August 2026)
- **Initial release**: 1.6.4518 (June 2025)
- **Expansion**: RimWorld – Odyssey (5th expansion, launched alongside 1.6)
- **Developer**: Ludeon Studios
- **Official changelog**: [Google Doc](https://docs.google.com/document/d/e/2PACX-1vRCjqVtPQDFGu4POiKTUd_8o3U2Asdhx99SOvcgU66ABdYtk3Cgndd53yJ6BC4tZX530pp_m6lf4Z9P/pub)
- **Official modder primer**: [Google Doc](https://docs.google.com/document/d/e/2PACX-1vRKE9u5ZW_zG45pxzwNvy4sxvozDeqtxlxpac5jwenOeW6liQCPgmPl9bIbtcMuqL1NPIDHOLFg64M_/pub)

## Key Changes in 1.6

### Performance Improvements
A major focus of 1.6 is performance optimization, especially for late-game colonies:
- Reworked many systems to spread out their workload
- Reduced overhead in commonly-called code paths
- Improved rendering and pathfinding performance

### New Features
- **Designator shapes system**: New vanilla system replacing `DraggableDimensions` with `DrawStyleCategory`
- **Gravship** (Odyssey DLC): Mobile flying base mechanic
- **New biomes**: Glowforest, Scarlands, Grasslands, Glacial plains, Lava fields
- **Landmarks system**: Procedural landscape variation
- **Caravan improvements**: Enhanced caravan mechanics
- **Quality of life**: Better base design tools, UI improvements

### Engine / Runtime
- **Unity version**: Upgraded to Unity 2022.3.35 (from 2021.x in 1.5)
- **Mono runtime**: Updated fork `unity-2022.3-mbe`
  - [GitHub: Unity-Technologies/mono/tree/unity-2022.3-mbe](https://github.com/Unity-Technologies/mono/tree/unity-2022.3-mbe)
- **.NET Framework**: Still net472 (Mono-compatible), no change to target framework

### Notable API Changes (relevant to modding)
- `DraggableDimensions` → `DrawStyleCategory` (DrawStyleCategoryDef)
- `<placingDraggableDimensions>` → `<drawStyleCategory>` in XML
- `<soundAmbient>` removed, replaced by `CompProperties_AmbientSound` ThingComp
- `<wildness>` moved from `RaceProperties` to a `StatDef` (now `<Wildness>` in `<statBases>`)
- New `<forceGender>` field in `RaceProperties`
- DamageDefs now have `<ignoreShields>` flag
- Various Hediff, Def, and UI changes

## Compatibility with Existing Mods

- Per Ludeon: "This update should be compatible with all savegames and mods" (referring to 1.6.4850 patch)
- However, many mods that patch specific IL code (transpilers) may need updates
- The modding community maintains a living document of changes at the [RimWorld Wiki 1.6 Mod Updates page](https://rimworldwiki.com/wiki/Modding_Tutorials/RimWorld_1.6_Mod_Updates)

## Community Resources

- **RimWorld Discord**: `#mod-development` channel for questions
- **RimWorld Wiki**: [Modding Tutorials](https://rimworldwiki.com/wiki/Modding_Tutorials)
- **RimRef (Krafs.Rimworld.Ref)**: NuGet package with RimWorld reference assemblies, updated daily from Steam
  - 1.6.4871 is the latest version (matches our game version 1.6.4850 closely)
  - [NuGet page](https://www.nuget.org/packages/Krafs.Rimworld.Ref/)
  - [GitHub: krafs/RimRef](https://github.com/krafs/RimRef)
- **Decompiled 1.6 Assembly-CSharp**: [GitHub: Dyyrlysh/RimworldDecompile](https://github.com/Dyyrlysh/RimworldDecompile) - source code exported via ILSpy for reference

## Local Game Installation

The game is installed at:
```
Q:\GAMES\RIMWORLD\Rimworld.v1.6.4850\game\
```

Game assemblies available at:
```
Q:\GAMES\RIMWORLD\Rimworld.v1.6.4850\game\RimWorldWin64_Data\Managed\
```

Key files:
- `Assembly-CSharp.dll` (15.7 MB) - Main game assembly
- `UnityEngine.dll` (125 KB) - Unity engine reference

These can be used for decompilation/reference when troubleshooting API changes.

## Steam Workshop

- Current mod (RimWorldAI Core): [Workshop ID 3269938006](https://steamcommunity.com/sharedfiles/filedetails/?id=3269938006)
- RimWorldAI Items: [Workshop ID 3269938552](https://steamcommunity.com/sharedfiles/filedetails/?id=3269938552)
