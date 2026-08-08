# Project Structure Changes for 1.6

This document details all structural changes needed to support RimWorld 1.6,
including .csproj modifications, About.xml updates, LoadFolders, and build scripts.

---

## 1. Multi-Version Support Strategy

### Option A: LoadFolders (Recommended)
Keep both 1.5 and 1.6 support using RimWorld's built-in versioned loading:

```
AICore/
├── 1.5/
│   └── Assemblies/
│       └── AICore.dll        (existing)
├── 1.6/
│   └── Assemblies/
│       └── AICore.dll        (new - compiled for 1.6)
├── About/
│   └── About.xml
├── LoadFolders.xml           (new - tells RimWorld which folder to use)
├── Languages/
├── Textures/
└── Libraries/
```

**LoadFolders.xml** example:
```xml
<?xml version="1.0" encoding="utf-8"?>
<loadFolders>
  <li IfVersion="1.5">1.5</li>
  <li IfVersion="1.6">1.6</li>
  <!-- Common resources always loaded -->
  <li>/</li>
</loadFolders>
```

This file goes at the mod root level and tells RimWorld which subfolder to load
assemblies from based on the game version.

### Option B: 1.6 Only
If we don't need to maintain 1.5 backwards compatibility, we can just:
- Update the existing `1.5/` folder to `1.6/`
- Update the .csproj target from `1.5` to `1.6`
- Only support 1.6

**Recommendation**: Start with Option B (1.6 only) since the fork is specifically
for 1.6 porting. Add LoadFolders later if needed.

---

## 2. .csproj Changes

### Current State (1.5)
- Project files: `AICore/Source/1.5.csproj`, `AIItems/Source/1.5.csproj`
- NuGet ref: `<PackageReference Include="Krafs.Rimworld.Ref" Version="1.5.*" />`
- Define constant: `RW15`
- ILRepack output: `../../1.5/Assemblies/`

### Target State (1.6)
Create new project files: `1.6.csproj` in each Source folder.

#### AICore/Source/1.6.csproj
```xml
<Project Sdk="Microsoft.NET.Sdk">
    <Import Project="Common.props" />
    <PropertyGroup>
        <ProjectGuid>{0067BAE0-20B4-4399-9BEE-7A1CC95DAACB}</ProjectGuid>
    </PropertyGroup>
    <PropertyGroup Condition="'$(Configuration)|$(Platform)'=='Release|AnyCPU'">
        <DefineConstants>RW16</DefineConstants>
        <DebugSymbols>false</DebugSymbols>
        <Optimize>true</Optimize>
        <DebugType>none</DebugType>
        <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
    </PropertyGroup>
    <PropertyGroup Condition="'$(Configuration)|$(Platform)'=='Debug|AnyCPU'">
        <DefineConstants>RW16;TRACE;DEBUG</DefineConstants>
        <DebugSymbols>true</DebugSymbols>
        <Optimize>false</Optimize>
        <DebugType>portable</DebugType>
        <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
    </PropertyGroup>
    <ItemGroup Label="Runtime">
        <PackageReference Include="Krafs.Rimworld.Ref" Version="1.6.*" />
    </ItemGroup>
</Project>
```

#### Key changes:
1. `Krafs.Rimworld.Ref` version: `1.5.*` → `1.6.*` (resolves to 1.6.4871)
2. Define constant: `RW15` → `RW16` (for conditional compilation if needed)
3. ILRepack output path: `../../1.5/Assemblies/` → `../../1.6/Assemblies/`
4. ZipMod target: `1.5/` → `1.6/` references

#### Common.props changes needed:
Update the `ZipMod` target:
- `<Dir15>..\1.5\</Dir15>` → `<Dir16>..\1.6\</Dir16>`
- `<Copy15 Include="..\1.5\**" />` → `<Copy16 Include="..\1.6\**" />`
- Output folder references updated

---

## 3. About.xml Changes

### AICore/About/About.xml
```xml
<ModMetaData>
  <name>RimWorldAI Core</name>
  <author>Trojan</author>
  <packageId>trojan.aicore</packageId>
  <modVersion>0.0.2.0</modVersion>  <!-- Bump version -->
  <supportedVersions>
    <li>1.6</li>                    <!-- Changed from 1.5 -->
  </supportedVersions>
  <loadAfter>
    <li>brrainz.harmony</li>
  </loadAfter>
  <modDependencies>
    <li>
      <packageId>brrainz.harmony</packageId>
      <displayName>Harmony</displayName>
      <steamWorkshopUrl>steam://url/CommunityFilePage/2009463077</steamWorkshopUrl>
      <downloadUrl>https://github.com/pardeike/HarmonyRimWorld/releases/latest</downloadUrl>
    </li>
  </modDependencies>
  <description>...</description>
</ModMetaData>
```

### AIItems/About/About.xml
Same pattern - change `<li>1.5</li>` to `<li>1.6</li>` in `<supportedVersions>`.

---

## 4. Directory.Build.props Changes

### AICore/Directory.Build.props
```xml
<Project>
    <PropertyGroup>
        <ModName>RimWorldAI Core</ModName>
        <ModFileName>AICore</ModFileName>
        <Repository>https://github.com/m0nk111/RWAI_Core</Repository>
        <ModVersion>0.0.2.0</ModVersion>
    </PropertyGroup>
</Project>
```

Note: Repository URL should point to the fork (m0nk111) if we're maintaining
separate repos, or stay as igoforth if we're just doing an internal fork.

---

## 5. Solution File (RWAILib.sln) Changes

Add new projects for 1.6:
```
Project("...") = "AI Core 1.6", "AICore\Source\1.6.csproj", "{NEW-GUID-1}"
EndProject
Project("...") = "AI Items 1.6", "AIItems\Source\1.6.csproj", "{NEW-GUID-2}"
EndProject
```

Keep or remove the 1.5 projects depending on multi-version strategy.

---

## 6. build.cake Changes

Update the Cake build script to target 1.6 projects:

```csharp
var aiCoreProject = File("./AICore/Source/1.6.csproj");  // Changed from 1.5
var aiItemsProject = File("./AIItems/Source/1.6.csproj");  // Changed from 1.5
```

---

## 7. Folder Structure

### New directory needed:
```
AICore/
└── 1.6/
    └── Assemblies/     (ILRepack output goes here)
AIItems/
└── 1.6/
    └── Assemblies/     (ILRepack output goes here)
```

### If using LoadFolders (multi-version):
```
RWAILib/
├── AICore/
│   ├── 1.5/
│   │   └── Assemblies/
│   ├── 1.6/                 ← NEW
│   │   └── Assemblies/      ← NEW (build output)
│   ├── About/
│   ├── Languages/
│   ├── Source/
│   │   ├── 1.5.csproj
│   │   ├── 1.6.csproj       ← NEW
│   │   ├── Common.props     (updated)
│   │   └── *.cs
│   ├── Libraries/
│   ├── Textures/
│   └── LoadFolders.xml      ← NEW (if multi-version)
├── AIItems/
│   ├── (same structure)
├── AIServer/                 (unchanged - Python, not version-bound)
├── bootstrap.py
├── build.cake
├── RWAILib.sln
└── docs/
    └── 1.6research/
```

---

## 8. Bootstrap.py Considerations

The `bootstrap.py` is version-agnostic (it downloads llamafile, models, etc.).
No changes needed to the Python bootstrap script for the 1.6 port.

However, if the bootstrap is downloaded from GitHub releases at runtime,
the release artifacts need to be updated or the C# code's `releaseString`
URL might need updating to point to the fork:

```csharp
// Current (in BootstrapTool.cs):
private const string releaseString = "https://api.github.com/repos/igoforth/RWAILib/releases/latest";
// If maintaining a separate fork:
private const string releaseString = "https://api.github.com/repos/m0nk111/RWAILib/releases/latest";
```

---

## 9. Steam Workshop Upload

For Steam Workshop upload:
1. The mod must be built and the `1.6/Assemblies/` folder populated
2. Use the RimWorld dev mode "Upload to Workshop" tool
3. Update the PublishedFileId.txt if creating a new listing, or keep existing
4. RimWorld 1.6 uses the same Steam Workshop upload process as 1.5
