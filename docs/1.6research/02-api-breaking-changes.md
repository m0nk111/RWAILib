# API Breaking Changes: 1.5 → 1.6

This document catalogs specific API changes between RimWorld 1.5 and 1.6 that are
relevant to the RWAILib codebase. Each change is assessed for impact on the mod.

---

## 1. Designators / Building Placement

### Change
- `DraggableDimensions` (virtual property) → `DrawStyleCategory` (returns `DrawStyleCategoryDef`)
- XML: `<placingDraggableDimensions>` → `<drawStyleCategory>`

### Impact on RWAILib: **NONE**
RWAILib does not interact with designators or building placement.

---

## 2. Sound Ambient

### Change
- `<soundAmbient>` removed from buildings
- Replaced by new `CompProperties_AmbientSound` ThingComp

### Impact on RWAILib: **NONE**
RWAILib does not use ambient sound properties.

---

## 3. Pawns / RaceProperties

### Change
- `<wildness>` moved from `RaceProperties` field to a `StatDef` in `<statBases>`
- Now capitalized as `<Wildness>`
- New `<forceGender>` field in `RaceProperties`

### Impact on RWAILib: **NONE**
RWAILib does not interact with pawn race properties.

---

## 4. DamageDefs

### Change
- New `<ignoreShields>` flag on DamageDefs
- Shield belts now check for both `<ignoreShields>` and `<isRanged>`

### Impact on RWAILib: **NONE**
RWAILib does not interact with damage definitions.

---

## 5. CompArt / Art System

### Status: **NEEDS INVESTIGATION**

The AIItems mod patches `CompArt.GenerateImageDescription` and accesses:
- `CompArt.titleInt` (private field)
- `CompArt.GenerateImageDescription()` method
- `ITab_Art.cachedImageDescription` (static field)
- `ITab_Art.cachedImageSource` (static field)
- `CompArt.taleRef` (property)
- `CompArt.Props.descriptionMaker` / `CompArt.Props.nameMaker`
- `TaleReference.GenerateText()`
- `TextGenerationPurpose.ArtDescription` / `TextGenerationPurpose.ArtName`
- `Scribe.saver.DebugOutputFor()`
- `GenText.CapitalizeAsTitle()`

### Risk Assessment
The art system is central to the mod's functionality. Any changes to:
- `CompArt.GenerateImageDescription()` signature or internals
- `ITab_Art` static cache fields
- `TaleReference.GenerateText()`
- `Scribe.saver.DebugOutputFor()`

...would require code changes.

### Mitigation
- Decompile `Assembly-CSharp.dll` from 1.6 to verify these APIs are unchanged
- Check the [Dyyrlysh/RimworldDecompile](https://github.com/Dyyrlysh/RimworldDecompile) repo for reference
- The mod uses `Krafs.Publicizer` to access private members, so private field name changes are the main risk

---

## 6. GlobalControls.GlobalControlsOnGUI

### Status: **HIGH RISK - Transpiler**

The AICore mod uses a Harmony **Transpiler** to inject code into `GlobalControls.GlobalControlsOnGUI`.
Transpilers are extremely sensitive to IL code changes.

### What the transpiler does
1. Searches for `GenUI.DrawTextWinterShadow` call
2. Finds `stloc.1` after it (with 4 instruction distance)
3. Searches for `GlobalControlsUtility.DoPlaySettings` call
4. Asserts screen height decrement pattern
5. Inserts status display code after position `j`

### Risk Assessment
If the IL code of `GlobalControls.GlobalControlsOnGUI` changed in 1.6 (which is likely
given UI changes and performance reworks), the transpiler will fail with
`"Cannot find GenUI.DrawTextWinterShadow in GlobalControls.GlobalControlsOnGUI!"`.

### Mitigation
- Decompile `GlobalControls.GlobalControlsOnGUI` in 1.6 and compare IL
- Update transpiler pattern matching logic
- Consider replacing transpiler with a Postfix patch if possible
- A Postfix on `GlobalControlsUtility.DoPlaySettings` might be simpler and more robust

---

## 7. MainMenuDrawer.MainMenuOnGUI

### Status: **MEDIUM RISK**

The AICore mod uses a Postfix on `MainMenuDrawer.MainMenuOnGUI` to display:
- Welcome dialog (first-run setup)
- Server status indicator

### Risk Assessment
- The method likely still exists but may have been refactored
- Changes to UI rendering pipeline in 1.6 could affect rendering
- The Postfix approach is more resilient than transpilers

### Mitigation
- Verify `MainMenuDrawer.MainMenuOnGUI` exists in 1.6
- Test that UI rendering works correctly

---

## 8. UIRoot.UIRootUpdate

### Status: **LOW RISK**

Postfix on `UIRoot.UIRootUpdate` for periodic task updates.

### Risk Assessment
- `UIRoot.UIRootUpdate` is a core lifecycle method unlikely to be removed
- Standard Postfix patch, very resilient

---

## 9. LongEventHandler.LongEventsOnGUI

### Status: **LOW RISK**

Postfix on `LongEventHandler.LongEventsOnGUI` for logging.

### Risk Assessment
- Core method, unlikely to change
- Standard Postfix, resilient

---

## 10. Current.Notify_LoadedSceneChanged

### Status: **LOW RISK**

Postfix on `Current.Notify_LoadedSceneChanged` for coroutine initialization.

### Risk Assessment
- Core lifecycle method
- Standard Postfix, resilient

---

## Summary Risk Matrix

| Patch Target | Patch Type | Risk Level | Action Required |
|---|---|---|---|
| `GlobalControls.GlobalControlsOnGUI` | Transpiler | 🔴 HIGH | Decompile & update IL pattern |
| `CompArt.GenerateImageDescription` | Postfix | 🟡 MEDIUM | Verify API unchanged |
| `MainMenuDrawer.MainMenuOnGUI` | Postfix | 🟡 MEDIUM | Verify method exists |
| `UIRoot.UIRootUpdate` | Postfix | 🟢 LOW | Minimal - test only |
| `LongEventHandler.LongEventsOnGUI` | Postfix | 🟢 LOW | Minimal - test only |
| `Current.Notify_LoadedSceneChanged` | Postfix | 🟢 LOW | Minimal - test only |

---

## Dependencies / NuGet Packages

| Package | 1.5 Version | 1.6 Version | Status |
|---|---|---|---|
| `Krafs.Rimworld.Ref` | `1.5.*` | `1.6.4871` (or `1.6.*`) | ✅ Available |
| `Lib.Harmony` | `2.3.3` | `2.3.3` | ✅ Same |
| `Google.Protobuf` | `3.27.0` | `3.27.0` | ✅ Same (not game-bound) |
| `Grpc` | `2.46.6` | `2.46.6` | ✅ Same (not game-bound) |
| `Contrib.Grpc.Core.M1` | `2.46.6` | `2.46.6` | ✅ Same (not game-bound) |
| `Krafs.Publicizer` | `2.2.1` | `2.2.1` | ✅ Same |
| `Newtonsoft.Json` | External DLL | External DLL | ✅ Same |
| `PolySharp` | `1.14.1` | `1.14.1` | ✅ Same |
| `ILRepack` | `2.0.33` | `2.0.33` | ✅ Same |
