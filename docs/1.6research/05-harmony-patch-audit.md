# Harmony Patch Audit: 1.5 → 1.6

## Audit Summary

Every Harmony patch in the RWAILib codebase, with 1.6 compatibility assessment
and remediation plan.

---

## AICore Patches (Source/Patches.cs)

### Patch 1: LongEventHandler.LongEventsOnGUI
**Type**: Postfix
**Purpose**: Flush log buffer during long event GUI rendering
**Code**:
```csharp
[HarmonyPatch(typeof(LongEventHandler), nameof(LongEventHandler.LongEventsOnGUI))]
public static class LongEventHandler_LongEventsOnGUI_Patch
{
    public static void Postfix() { LogTool.Log(); }
}
```

**1.6 Risk**: 🟢 LOW
- `LongEventHandler.LongEventsOnGUI` is a core logging/GUI method
- Unlikely to be renamed or removed
- Postfix is resilient to internal changes
**Action**: None needed. Verify at runtime.

---

### Patch 2: MainMenuDrawer.MainMenuOnGUI
**Type**: Postfix (with static constructor)
**Purpose**: Display welcome dialog (first-run) and server status indicator at main menu
**Code**:
```csharp
[StaticConstructorOnStartup]
[HarmonyPatch(typeof(MainMenuDrawer), nameof(MainMenuDrawer.MainMenuOnGUI))]
public static class MainMenuDrawer_MainMenuOnGUI_Patch
{
    // Loads banner, model sizes, handles welcome dialog UI
    // Draws server status overlay
    public static void Postfix() { Welcome(); ShowStatus(); }
}
```

**Accessed APIs**:
- `AICoreMod.self.Content.ModMetaData.PreviewImagePath` - path to preview image
- `UI.screenWidth`, `UI.screenHeight` - screen dimensions
- `Widgets.DrawBoxSolidWithOutline()` - UI rendering
- `Widgets.DrawTexturePart()` - image rendering
- `Listing_Standard` - UI layout
- `Find.WindowStack.Add()` - window management
- `UpdateLanguage.activeLanguage` - custom tracking
- `Graphics.ButtonServerStatus` - custom texture array

**1.6 Risk**: 🟡 MEDIUM
**Reasoning**:
- `MainMenuDrawer.MainMenuOnGUI` likely exists but may be refactored
- UI rendering APIs (`Widgets`, `Find.WindowStack`) are stable
- The `PreviewImagePath` property access chain could change
**Action**:
1. Verify `MainMenuDrawer.MainMenuOnGUI` exists in 1.6 (decompile Assembly-CSharp.dll)
2. Test that `ModMetaData.PreviewImagePath` still returns valid path
3. Run and check that welcome dialog renders correctly

---

### Patch 3: GlobalControls.GlobalControlsOnGUI (TRANSPILER)
**Type**: Transpiler (IL code injection)
**Purpose**: Inject server status display into the in-game bottom-right controls area
**Code**:
```csharp
[HarmonyPatch(typeof(GlobalControls), nameof(GlobalControls.GlobalControlsOnGUI))]
public static class GlobalControls_GlobalControlsOnGUI_Patch
{
    public static IEnumerable<CodeInstruction> Transpiler(IEnumerable<CodeInstruction> instructions)
    {
        // 1. Find GenUI.DrawTextWinterShadow call
        // 2. Find stloc.1 after it (4 instruction gap)
        // 3. Find GlobalControlsUtility.DoPlaySettings call
        // 4. Assert screen height decrement pattern
        // 5. Insert: call PlayGUIServerStatusPatch(ref float num2)
        //            then duplicate the height decrement
    }
}
```

**IL Pattern Expected**:
```
... call GenUI.DrawTextWinterShadow ...
... ldloc.x, ldc.r4, sub, stloc.y ...  (height decrement)
... ldloca.s, call GlobalControlsUtility.DoPlaySettings ...
```

**1.6 Risk**: 🔴 HIGH
**Reasoning**:
- Transpilers are extremely fragile — any IL reordering breaks them
- 1.6 has performance reworks which may have changed method internals
- The `GlobalControls.GlobalControlsOnGUI` method is a prime candidate for refactoring
- Even if the method exists, the internal IL flow may differ
**Mitigation Options**:
1. **Decompile & update**: Decompile `GlobalControls.GlobalControlsOnGUI` in 1.6,
   compare IL, update pattern matching logic
2. **Replace with Postfix**: A Postfix on `GlobalControls.GlobalControlsOnGUI` that
   draws the status indicator after the original method completes. This is simpler:
   ```csharp
   [HarmonyPatch(typeof(GlobalControls), nameof(GlobalControls.GlobalControlsOnGUI))]
   public static class GlobalControls_GlobalControlsOnGUI_Patch
   {
       public static void Postfix()
       {
           // Draw status indicator at bottom-right
           // No transpiler needed - just draw after original
       }
   }
   ```
3. **Use GlobalControlsUtility.DoPlaySettings Postfix**:
   The transpiler was inserting into the play settings area. A Postfix on
   `GlobalControlsUtility.DoPlaySettings` might achieve the same result more robustly.

**Recommended Action**: Replace transpiler with Postfix approach (Option 2 or 3).
This eliminates the fragile IL pattern matching entirely.

---

### Patch 4: Current.Notify_LoadedSceneChanged
**Type**: Postfix + static constructor (in Main.cs)
**Purpose**: Initialize coroutine processors when scene loads
**Code**:
```csharp
[HarmonyPatch(typeof(Current), nameof(Current.Notify_LoadedSceneChanged))]
[StaticConstructorOnStartup]
public static class Main
{
    public static void Postfix()
    {
        if (GenScene.InEntryScene) _ = Current.Root_Entry.StartCoroutine(Process());
        if (GenScene.InPlayScene) _ = Current.Root_Play.StartCoroutine(Process());
    }
}
```

**1.6 Risk**: 🟢 LOW
- `Current.Notify_LoadedSceneChanged` is a core lifecycle method
- `GenScene.InEntryScene` / `GenScene.InPlayScene` are stable
- `Current.Root_Entry` / `Current.Root_Play` are core references
**Action**: None needed. Verify at runtime.

---

### Patch 5: UIRoot.UIRootUpdate
**Type**: Postfix
**Purpose**: Run periodic update tasks (server status check, job monitoring, language update)
**Code**:
```csharp
[HarmonyPatch(typeof(UIRoot), nameof(UIRoot.UIRootUpdate))]
public static partial class GenerallTimeUpdates_Patch
{
    private static readonly Dictionary<string, UpdateTaskTime> updateTasks = new()
    {
        { "ServerStatus", new UpdateTaskTime(() => 2.0f, UpdateServerStatus.Task, false) },
        { "MonitorJobs", new UpdateTaskTime(() => 5.0f, UpdateJobStatus.Task, false) },
        { "UpdateLanguage", new UpdateTaskTime(() => 5.0f, UpdateLanguage.Task, true) },
    };
    public static void Postfix() { /* run tasks based on time */ }
}
```

**1.6 Risk**: 🟢 LOW
- `UIRoot.UIRootUpdate` is a core Unity lifecycle method
- Postfix is resilient
**Action**: None needed. Verify at runtime.

---

## AIItems Patches (Source/Patches.cs)

### Patch 6: LongEventHandler.LongEventsOnGUI
**Type**: Postfix (identical to AICore's)
**1.6 Risk**: 🟢 LOW — Same as Patch 1.

---

### Patch 7: CompArt.GenerateImageDescription ⭐
**Type**: Postfix
**Purpose**: Replace artwork title and description with AI-generated text
**Code**:
```csharp
[HarmonyPatch(typeof(CompArt), nameof(CompArt.GenerateImageDescription))]
public static class CompArt_GenerateImageDescription_Patch
{
    public static void Postfix(CompArt __instance, ref TaggedString __result)
    {
        // Hash item ID
        // Check cache status (Done/NotDone/Working)
        // If Done: replace __instance.titleInt and __result with cached values
        //         Also update ITab_Art.cachedImageDescription
        // If NotDone: generate default text, submit AI job
        // If Working: do nothing (wait for AI response)
    }
}
```

**Accessed APIs**:
| API | Access Method | Risk |
|-----|--------------|------|
| `CompArt.GenerateImageDescription` | Patch target | Medium - verify method exists |
| `CompArt.parent.ThingID` | Public property | Low |
| `CompArt.titleInt` | Private field (via Publicizer) | Medium - private field name could change |
| `CompArt.taleRef` | Property | Medium |
| `CompArt.Props.descriptionMaker` | Property | Medium |
| `CompArt.Props.nameMaker` | Property | Medium |
| `TaleReference.GenerateText()` | Method call | Medium |
| `TextGenerationPurpose.ArtDescription` | Enum value | Low |
| `TextGenerationPurpose.ArtName` | Enum value | Low |
| `ITab_Art.cachedImageDescription` | Static field (via Publicizer) | Medium - static защ field name |
| `ITab_Art.cachedImageSource` | Static field (via Publicizer) | Medium |
| `Scribe.saver.DebugOutputFor()` | Method call | Low-Medium |
| `GenText.CapitalizeAsTitle()` | Method call | Low |

**1.6 Risk**: 🟡 MEDIUM
**Reasoning**:
- The art system is core gameplay, unlikely to be fundamentally redesigned
- However, private field names (`titleInt`, `cachedImageDescription`, `cachedImageSource`)
  could be renamed during refactoring
- `Krafs.Publicizer` generates `IgnoresAccessChecksToAttribute` to access privates
- The Publicizer auto-generates at build time based on current assemblies, so if a
  private field is renamed, the build will fail (not a silent runtime error)
**Action**:
1. Build against 1.6 Krafs.Rimworld.Ref and check for compile errors
2. If `titleInt` etc. are renamed, update references in code
3. Run and verify art descriptions are replaced correctly

---

## Summary

| # | Patch | Type | Component | Risk | Action |
|---|-------|------|-----------|------|--------|
| 1 | `LongEventHandler.LongEventsOnGUI` | Postfix | AICore | 🟢 LOW | Test only |
| 2 | `MainMenuDrawer.MainMenuOnGUI` | Postfix | AICore | 🟡 MEDIUM | Verify method exists |
| 3 | `GlobalControls.GlobalControlsOnGUI` | Transpiler | AICore | 🔴 HIGH | **Replace with Postfix** |
| 4 | `Current.Notify_LoadedSceneChanged` | Postfix | AICore | 🟢 LOW | Test only |
| 5 | `UIRoot.UIRootUpdate` | Postfix | AICore | 🟢 LOW | Test only |
| 6 | `LongEventHandler.LongEventsOnGUI` | Postfix | AIItems | 🟢 LOW | Test only |
| 7 | `CompArt.GenerateImageDescription` | Postfix | AIItems | 🟡 MEDIUM | Build check + test |

### Priority Actions
1. **Replace transpiler** in `GlobalControls_GlobalControlsOnGUI_Patch` with a Postfix
2. **Build against 1.6** ref assemblies and fix any compile errors
3. **Test runtime** behavior of all patches
