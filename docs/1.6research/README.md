# RimWorld 1.6 Research Documentation

This directory contains comprehensive research findings for porting the RWAILib framework
from RimWorld 1.5 to RimWorld 1.6 (Odyssey).

## Documents

| Document | Description |
|----------|-------------|
| [01-rimworld-1.6-overview.md](01-rimworld-1.6-overview.md) | High-level overview of RimWorld 1.6, the Odyssey DLC, and what changed |
| [02-api-breaking-changes.md](02-api-breaking-changes.md) | Specific API changes between 1.5 and 1.6 that affect this mod |
| [03-project-structure-changes.md](03-project-structure-changes.md) | .csproj, NuGet, LoadFolders, and About.xml changes needed |
| [04-codebase-analysis.md](04-codebase-analysis.md) | Analysis of the current RWAILib codebase and what needs porting |
| [05-harmony-patch-audit.md](05-harmony-patch-audit.md) | Audit of every Harmony patch and transpiler, with 1.6 risk assessment |
| [06-debugging-and-testing.md](06-debugging-and-testing.md) | Dev mode, debugging tools, and testing workflow for 1.6 |
| [07-porting-plan.md](07-porting-plan.md) | Step-by-step implementation plan for the 1.6 port |
| [08-handover-for-next-agent.md](08-handover-for-next-agent.md) | Context handoff document for the next AI agent |
| [09-rimbridgeserver-testing.md](09-rimbridgeserver-testing.md) | How to use RimBridgeServer for live testing of the 1.6 port |

## Quick Summary

- **Game version**: 1.6.4850 (latest as of August 2026)
- **Mod currently supports**: 1.5 only
- **Krafs.Rimworld.Ref**: 1.6.4871 available on NuGet ✅
- **Harmony (Lib.Harmony)**: 2.3.3 still compatible ✅
- **.NET target**: net472 (unchanged) ✅
- **Unity version**: Upgraded to 2022.3.35 in 1.6
- **Main risks**: Transpiler IL changes, CompArt API changes, GlobalControls restructuring
