# Codebase Analysis: RWAILib

## Overview

RWAILib is a framework that runs a small language model (Microsoft Phi-3) entirely
locally and wires it into RimWorld. It has three components:

| Component | Language | Lines of Code (approx) | Role |
|-----------|----------|------------------------|------|
| AICore | C# (Harmony) | ~3,200 | Framework: bootstraps, launches, supervises Python server; gRPC channel; settings UI |
| AIItems | C# (Harmony) | ~1,200 | Feature mod: hooks CompArt to replace artwork text with AI-generated text |
| AIServer | Python 3.11 | ~1,700 | gRPC server driving local llamafile/Phi-3 instance |

## Architecture

```
User views art in RimWorld
        ↓
AIItems: CompArt.GenerateImageDescription Postfix
        ↓  (hashes item ID, checks cache)
        ↓  (if not cached: submits JobRequest with XML def, title, description)
        ↓
AICore: JobClient (gRPC :50051)
        ↓
AIServer: Python gRPC server
        ↓  (prompts local Phi-3 model via llamafile HTTP :50052)
        ↓  (returns new title + description in JobResponse)
        ↓
AICore: JobClient → callback
        ↓
AIItems: UpdateItemDescriptions (caches result, updates CompArt fields)
        ↓
User sees AI-generated art description
```

## AICore Source Files

| File | LOC | Purpose |
|------|-----|---------|
| `Main.cs` | ~188 | Mod entry point. Harmony patch on `Current.Notify_LoadedSceneChanged`. Starts coroutines, initializes settings, manages lifecycle. |
| `Patches.cs` | ~497 | All Harmony patches: MainMenu drawer (welcome + status), GlobalControls transpiler (status in-game), UIRoot update loop. |
| `BootstrapTool.cs` | ~826 | Downloads and manages Python runtime, llamafile, AI model. Handles GPU/VRAM detection, gRPC native lib setup, update checking. |
| `ServerManager.cs` | ~249 | Manages the Python AIServer process lifecycle (start/stop/restart). |
| `JobClient.cs` | ~199 | gRPC client. Sends job requests to AIServer, receives responses. |
| `Settings.cs` | ~364 | Mod settings: enable/disable, model size, auto-update check. Settings UI. |
| `Tools.cs` | ~371 | Utility functions: SafeWait, logging, hash helpers. |
| `UX.cs` | ~232 | UI rendering helpers. |
| `LogTool.cs` | ~136 | Logging utility with queue/buffer mechanism. |
| `LanguageMapping.cs` | ~146 | Maps RimWorld SupportedLanguage enum to bootstrap.py language args. |
| `Lanczos.cs` | ~134 | Lanczos resampling for image downscaling (welcome banner). |
| `ProcessInterruptHelper.cs` | ~200 | Cross-platform process interruption (SIGINT on Unix, taskkill on Windows). |
| `Defs.cs` | ~22 | Def definitions. |
| `Tasks.cs` | ~67 | Update task definitions (server status monitoring, job monitoring, language update). |
| `UpdateTask.cs` | ~32 | Update task time tracking. |
| `UnityWebRequestException.cs` | ~13 | Custom exception for Unity web request errors. |

### AICore Unused Files (in `Source/unused/`)
~1,600 LOC of experimental code: async pools, thread managers, coroutine managers,
multi-API, Lanczos expanded, log sinks, type resolvers, work queues.

## AIItems Source Files

| File | LOC | Purpose |
|------|-----|---------|
| `Patches.cs` | ~135 | Core: Harmony Postfix on `CompArt.GenerateImageDescription`. Hashes item, checks cache, submits jobs, replaces text. |
| `Tasks.cs` | ~202 | `UpdateItemDescriptions`: manages job queue, cache dictionary, status tracking. |
| `CompressedDictionary.cs` | ~237 | Memory-efficient dictionary for caching item descriptions. |
| `CityHash.cs` | ~1K | CityHash implementation for fast item ID hashing. |
| `Tools.cs` | ~371 | Shared utilities (mirrors AICore). |
| `UX.cs` | ~232 | Shared UI helpers (mirrors AICore). |
| `LogTool.cs` | ~136 | Shared logging (mirrors AICore). |
| `Settings.cs` | ~17 | Minimal settings. |
| `Main.cs` | ~147 | Mod entry point (mirrors AICore). |

## AIServer Source Files

| File | LOC | Purpose |
|------|-----|---------|
| `__init__.py` | ~311 | Server entry point. Main function. |
| `templates.py` | ~540 | Prompt templates for different languages and job types. |
| `client.py` | ~380 | Llamafile HTTP client. Manages model loading and inference. |
| `server.py` | ~108 | gRPC server setup and request handling. |
| `health.py` | ~51 | Health check endpoint. |
| `job/job_pb2.py` | ~39 | Generated protobuf code. |
| `job/job_pb2_grpc.py` | ~103 | Generated gRPC service code. |
| `job/job_pb2.pyi` | ~111 | Type stubs for protobuf. |
| `loader.py` | ~338 | Self-contained .pyz loader (platform-specific Python + library loading). |
| `protos/job.proto` | ~83 | gRPC service definition. |

## Key Technical Details

### gRPC Protocol
- Service: `JobManager.JobService(JobRequest) returns (JobResponse)`
- Uses `oneof` for extensible job types (currently only `ArtDescriptionJob`)
- Port: 50051 (gRPC)
- Port: 50052 (llamafile HTTP)

### Native Libraries
The mod ships compressed gRPC native libraries in `Libraries/`:
- `grpc_csharp_ext.x64.dll.gz` (Windows x64)
- `grpc_csharp_ext.x86.dll.gz` (Windows x86)
- `libgrpc_csharp_ext.x64.dylib.gz` (macOS x64)
- `libgrpc_csharp_ext.arm64.dylib.gz` (macOS ARM64)
- `libgrpc_csharp_ext.x64.so.gz` (Linux x64)

These are decompressed at startup and copied to RimWorld's `Mono/` directory.
A custom MSBuild task handles this compression during build.

### ILRepack
All dependencies (gRPC, protobuf, Newtonsoft.Json, etc.) are merged into a
single DLL via ILRepack. The build process:
1. Compiles the mod
2. Publicizes game assemblies (via Krafs.Publicizer)
3. Copies publicized assemblies to `shared/`
4. Moves native gRPC libs to `Libraries/` and compresses them
5. ILRepack merges all managed DLLs into one
6. Output goes to `../../1.5/Assemblies/` (→ `1.6/Assemblies/` for 1.6)

### Conditional Compilation
- `RW15` define constant for 1.5-specific code
- `RW16` define constant for 1.6-specific code (to be added)
- `DEBUG` / `TRACE` for debug builds

### Bootstrap Flow (first run)
1. AICore detects hardware (platform, architecture, VRAM)
2. Shows welcome dialog with model size selection
3. Downloads cosmopolitan curl binary
4. Downloads llamafile (~35 MB)
5. Downloads AIServer.pyz (~103 MB)
6. Downloads Phi-3 model (2.4–8.6 GB depending on size)
7. Writes `.version` file
8. Starts AIServer, connects gRPC client

### Update Flow (subsequent runs)
1. Checks `.version` against GitHub releases API
2. If newer version available, re-downloads AIServer.pyz
3. If already current, starts server directly
