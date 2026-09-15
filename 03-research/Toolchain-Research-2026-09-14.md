---
title: Toolchain — Research
date: 2026-09-14
status: research-draft
tags: [phoenix, research, unreal, xcode, macos, toolchain]
---

# Toolchain — Research

> [!info] **Scope.** Can this Mac compile Unreal C++ at all, and which Unreal Engine + Xcode
> combinations are candidates? **Facts only — no decision.** macOS only; Windows is out of scope.

> [!danger] **Headline.** UE 5.5.4 + Xcode 26.6 **cannot compile C++ on this machine.** Tested,
> not inferred: `Platform Mac is not a valid platform to build.`, exit 6. The installed engine has
> no officially supported Xcode for this macOS version.

## Revision history

| rev | date | what changed |
|---|---|---|
| r1 | 2026-09-14 | First draft. |
| r2 | 2026-09-14 | Added §3.4: why an ordinary C++ compiler does not bypass the block. |
| r5 | 2026-09-14 | Editor reaches Project Browser with Xcode 26.6: 0 errors, 4 warnings, including a fallback from Shader Model 6 to 5 (§6c). |
| r4 | 2026-09-14 | UE 5.8.2 installed and tested: SDK `Valid` and C++ compile succeeds with the existing Xcode 26.6 (§6a). Editor launch failed on a missing Metal toolchain; installed and verified (§6b). |
| r3 | 2026-09-14 | Combination D chosen — see [[Decisions]] ADR-006. UE 5.8 release confirmed (June 17, 2026). |

## Evidence tiers

| tag | meaning |
|---|---|
| **[measured]** | Executed on this machine; output quoted verbatim |
| **[source]** | Read in the installed engine's files |
| **[doc]** | Official Epic or Apple documentation, fetched 2026-09-14 |
| **[forum]** | Community report — weakest tier, not relied on |
| **[NOT verified]** | See §7 |

---

## 1. This machine

**[measured]**

| component | value | how |
|---|---|---|
| macOS | **26.6.2** (Tahoe), build 25G83 | `sw_vers` |
| Xcode | **26.6**, build 17F113 — the only Xcode installed | `xcodebuild -version`; `mdfind` for `com.apple.dt.Xcode` |
| Unreal Engine | **5.5.4**, changelist 40574608, launcher (binary) build | `Engine/Build/Build.version` |
| Engine path | `/Users/Shared/Epic Games/UE_5.5` | |
| git-lfs | 3.8.0 | `git lfs version` |

## 2. What UE 5.5 accepts

**[source]** `Engine/Config/Apple/Apple_SDK.json`:

```json
"MainVersion": "15.2",
"MinVersion": "15.2.0",
"MaxVersion": "16.9.0",
```

**[doc]** Epic's macOS requirements page lists UE 5.5 as: Xcode 15.2 minimum, "15.4 or newer"
recommended; macOS Ventura 13.5 minimum. The page gives no maximum; the engine file does.

## 3. The engine's own verdict — two tests

### 3.1 SDK check

**[measured]** `RunUAT.sh Turnkey -command=VerifySdk -platform=Mac -UpdateIfNeeded=false`:

```
Installed Sdk validity:
Mac: (Status=Invalid, MinAllowed_Sdk=15.2.0, MaxAllowed_Sdk=16.9.0, Current_Sdk=26.6, Allowed_AutoSdk=15.2, Current_AutoSdk=, Flags="InstalledSdk_InvalidVersionExists, Platform_ValidHostPrerequisites")
```

### 3.2 An actual compile

**[measured]** Epic's `TP_Blank` C++ template, copied to a throwaway scratch directory (not the
Phoenix project), built with
`Engine/Build/BatchFiles/Mac/Build.sh TP_BlankEditor Mac Development -Project=<copy>/TP_Blank.uproject`:

```
Creating makefile for TP_BlankEditor (no existing makefile)
Total execution time: 3.49 seconds
Platform Mac is not a valid platform to build. Check that the SDK is installed properly and that you have the necessary platform support files (DataDrivenPlatformInfo.ini, SDK.json, etc).
```

Exit code **6**. No compiler was invoked; the build tool rejected the platform before compiling.

### 3.3 Why — the code path

**[source]**

```
UnrealBuildTool/Configuration/UEBuildPlatform.cs:625
    return BuildPlatformDictionary.ContainsKey(Platform)
        && (bIgnoreSDKCheck || BuildPlatformDictionary[Platform].HasRequiredSDKsInstalled() == SDKStatus.Valid);
```

```
UnrealBuildTool/Platform/Mac/ApplePlatformSDK.cs:38
    if (Status == SDKStatus.Invalid && !RuntimePlatform.IsMac && Unreal.IsBuildMachine())
    {
        Status = SDKStatus.Valid;
    }
```

An out-of-range Xcode is `Invalid`, so the Mac platform is not buildable. The only override applies to
**non-Mac build machines**; it never fires here.

### 3.4 Why "use an ordinary C++ compiler" does not get around it

**[measured]** The machine already has a standard C++ compiler, and it is *newer* than UE 5.5 expects:

```
$ xcrun clang++ --version
Apple clang version 21.0.0 (clang-2100.1.1.101)
$ g++ --version            # on macOS, g++ is Apple clang
Apple clang version 21.0.0 (clang-2100.1.1.101)
```

`Apple_SDK.json`'s own version table stops at `"16.0.0-17.0.6"` (Xcode 16 → LLVM 17.0.6). It has no
entry for this compiler.

**[source]** UnrealBuildTool chooses the compiler and SDK itself, from Xcode's directory:

```
ToolChain/AppleToolChain.cs:75
    ToolchainDir = DirectoryReference.Combine(XcodeDeveloperDir, "Toolchains/XcodeDefault.xctoolchain/usr/bin");
ToolChain/AppleToolChain.cs:76
    SDKDir = DirectoryReference.Combine(XcodeDeveloperDir, $"Platforms/{OSPrefix}.platform/Developer/SDKs/{OSPrefix}.sdk");
Platform/Mac/MacToolChain.cs:63
    private const string MacCompiler = "clang++";
Platform/Mac/MacToolChain.cs:206
    Arguments.Add($"-isysroot \"{Settings.GetSDKPath()}\"");
```

Three consequences:

1. **The failure in §3.2 happens before any compiler runs.** It is a version check in UnrealBuildTool, so
   a different compiler never gets invoked.
2. **Compiling without UnrealBuildTool means rebuilding what it does:** running UnrealHeaderTool to
   generate `*.generated.h`, computing each module's include paths and defines, linking against the
   prebuilt engine libraries, and writing the module manifest the editor checks. Any compiler also still
   needs Apple's macOS SDK (Cocoa, Metal and other frameworks).
3. **The version cap has a concrete technical reason.** `ToolChain/ClangToolChain.cs:599-600` adds
   `-Wall` and `-Werror` to every compile, so any warning becomes an error. Newer compilers add new
   warnings, so engine headers that compiled cleanly on clang 17 can fail on clang 21. **[NOT verified]**
   for UE 5.5 on this compiler; see option F in §6.

**[source]** The accepted range is read from the engine directory
(`UEBuildPlatformSDK.cs:1051`, `LoadJsonFile` → `Unreal.EngineDirectory`). No project-level override was
found; that search was not exhaustive.

## 4. Why "just install an older Xcode" is not straightforward

**[doc]** Apple's Xcode system-requirements table:

| Xcode | supported macOS | this Mac (26.6.2) | inside UE 5.5's range? |
|---|---|---|---|
| 16.4 | Sequoia 15.3 – **Tahoe 26.1.x** | ✗ outside | ✓ |
| 16.3 | Sequoia 15.2 – Sequoia 15.x | ✗ outside | ✓ |
| 16.2 / 16.1 / 16 | Sonoma 14.5 – Sequoia 15.x | ✗ outside | ✓ |
| 26.1.1 | Sequoia 15.6 – Tahoe 26.x | ✓ inside | ✗ |
| 26.6 *(installed)* | Tahoe 26.2 – Tahoe 26.x | ✓ inside | ✗ |

**No Xcode version is inside both UE 5.5's accepted range and Apple's supported range for this
macOS.** Xcode 16.4 is the nearest, and Apple stops supporting it at Tahoe 26.1.x.

Whether Xcode 16.4's command-line toolchain *works anyway* on 26.6.2, despite being unsupported, is
not established — see §7.

## 5. Newer Unreal Engine versions

**[doc]** Epic's macOS requirements page (UE 5.8 documentation):

| UE | Xcode minimum | Xcode recommended | macOS minimum | note on the page |
|---|---|---|---|---|
| **5.8** | 26.0 | **26.1.1** | Sonoma 14.5 | "Xcode 26.4 is not compatible with Unreal Engine" |
| 5.6 – 5.7 | 15.2 | 15.4 or newer | Sonoma 14.0 | — |
| 5.5 | 15.2 | 15.4 or newer | Ventura 13.5 | — |

**[forum]** A report of UE 5.6.1 and 5.7 failing to build on macOS 26.1 with Xcode 26.1.1; no working
combination confirmed in the thread.

## 6. Candidate combinations

| # | combination | Epic | Apple | tested here |
|---|---|---|---|---|
| A | UE 5.5.4 + Xcode 26.6 *(current)* | ✗ above max 16.9.0 | ✓ | **✗ FAILS** (§3) |
| B | UE 5.5.4 + Xcode 16.4, side by side | ✓ in range | ✗ unsupported on 26.6.2 | not tested |
| C | UE 5.6 / 5.7 + any Xcode 26.x | max version unknown; forum reports failures | ✓ | not tested |
| D | **UE 5.8 + Xcode 26.1.1**, side by side | ✓ recommended | ✓ supported on Tahoe 26.x | not tested |
| E | UE 5.8 + Xcode 26.6 *(installed)* | 26.4 flagged incompatible; 26.5 and 26.6 not stated | ✓ | not tested |
| F | UE 5.5.4 with `MaxVersion` edited in `Apple_SDK.json` | unsupported; modifies the shared engine install | ✓ | not tested |

**Only D is documented as supported by both Epic and Apple.** B, E and F each rely on something one
of the vendors does not support.

## 6a. UE 5.8.2 on this machine — tested (r4)

**[measured]** Installed from the Epic Games Launcher: **UE 5.8.2**, changelist 56702186, 43 GB, at
`/Users/Shared/Epic Games/UE_5.8`. `TP_Blank` and `TP_TopDown` templates present.

**[source]** UE 5.8.2 `Engine/Config/Apple/Apple_SDK.json`:

```json
"MainVersion": "26.1.1",
"MinVersion": "15.2.0",
"MaxVersion": "27.9.0",
```

Its version table now reaches `"26.4.0-21.1.6"` and `"27.0.0-21.1.6"`.

**[measured]** SDK check with the **existing Xcode 26.6**:

```
Mac: (Status=Valid, MinAllowed_Sdk=15.2.0, MaxAllowed_Sdk=27.9.0, Current_Sdk=26.6, Allowed_AutoSdk=26.1.1, Current_AutoSdk=, Flags="InstalledSdk_ValidVersionExists")
```

**[measured]** C++ compile of `TP_Blank` (throwaway copy) with UE 5.8.2 + Xcode 26.6:

```
[1/5] Compile [Apple] SharedPCH.UnrealEd.Project.ValApi.ValExpApi.Cpp20.h
[2/5] Compile [Apple] TP_Blank.cpp
[3/5] Compile [Apple] PerModuleInline.gen.cpp
[4/5] Link [Apple] libUnrealEditor-TP_Blank.dylib
Result: Succeeded
Total execution time: 31.70 seconds
```

Exit 0; `libUnrealEditor-TP_Blank.dylib` produced. **Combination E (UE 5.8 + Xcode 26.6) compiles C++.**

> [!warning] **Doc-vs-reality delta.** Epic's requirements page states *"Xcode 26.4 is not compatible
> with Unreal Engine"*, yet UE 5.8.2's own `Apple_SDK.json` accepts up to 27.9.0, and a real compile with
> Xcode 26.6 succeeded. The page may predate 5.8.2, or the incompatibility may be at runtime rather than
> at compile time. **Editor runtime with 26.6 is not yet verified** (see §6b).

## 6b. Editor launch — Metal toolchain missing (r4)

**[measured]** Launching `UnrealEditor` 5.8.2 stopped at "0% – Initializing.." with:

```
Xcode Metal Compiler error: error: error: cannot execute tool 'metal' due to missing Metal Toolchain;
use: xcodebuild -downloadComponent MetalToolchain
```

Confirmed from the shell:

```
$ xcrun metal --version
error: error: cannot execute tool 'metal' due to missing Metal Toolchain; use: xcodebuild -downloadComponent MetalToolchain
$ xcodebuild -showComponent MetalToolchain
Build Version: 17F109
Status: uninstalled
```

**Cause:** in Xcode 26.6, `Toolchains/XcodeDefault.xctoolchain/usr/bin/metal` exists but is a stub; the Metal
shader compiler is a **separately downloaded component**. Unreal needs it to compile Metal shaders at editor
startup. The C++ compile in §6a does not use it, which is why that passed.

**Fix, applied 2026-09-14:** `xcodebuild -downloadComponent MetalToolchain` — 687.9 MB, 72 s, exit 0.

```
$ xcodebuild -showComponent MetalToolchain
Build Version: 17F109
Status: installed
Toolchain Identifier: com.apple.dt.toolchain.Metal.32023.883
$ xcrun metal --version
Apple metal version 32023.883 (metalfe-32023.883)
```

**Failure mode worth remembering:** a successful C++ build does not prove the editor can start. Shader
compilation is a second toolchain dependency.

## 6c. Editor start after the Metal fix (r5)

**[measured]** UE 5.8.2 editor relaunched and reached the **Project Browser**. Log:
`~/Library/Logs/Unreal Engine/Editor/Unreal.log`.

```
LogInit: Engine Version: 5.8.2-56702186+++UE5+Release-5.8
LogModuleManager: InternalLoadLibrary: 'MetalRHI' (...libUnrealEditor-MetalRHI.dylib)
LogRHI: Texture pool is 17203 MB (70% of 24576 MB)
LogShaderCompilers: Display: Using 7 local workers for shader compilation
```

**0 `Error:` lines, 4 `Warning:` lines**, all four read:

| warning | meaning | blocks Step 1? |
|---|---|---|
| `LogMetal: Warning: To use SM6 on this system, please ensure you are running Mac OS 15. Falling back to SM5` | The editor runs **Shader Model 5, not 6**. The message asks for macOS 15, but this machine runs macOS 26.6.2, which is newer, so the check appears not to recognise macOS 26. **Why, and which features need SM6: [NOT verified].** | No — click-to-move and a basic mesh do not need it |
| `LogAudioMixerAudioUnit: Warning: Error querying Sample Rate: 2003332927` | `2003332927` = `0x77686F3F` = the four characters `who?`, a Core Audio status code (believed to be "unknown property"; name not verified). Audio device query failed. | No — no audio in Step 1 |
| `LogAppleWebBrowser: Warning: SetWebBrowserVisibility 0!` / `1!` | The Project Browser's embedded web panel toggling | No |

**What this does and does not prove:** the editor starts, loads the Metal renderer, and runs shader
compilation with Xcode 26.6. It does **not** yet prove that a C++ project opens in the editor, or that a level
plays. That is proven by the first Phoenix compile-and-open.

## 7. NOT verified

| claim | why unverified | what would settle it |
|---|---|---|
| UE 5.8's `Apple_SDK.json` `MaxVersion` | UE 5.8 is not installed | Install UE 5.8, read `Engine/Config/Apple/Apple_SDK.json`, run the §3 tests |
| ~~UE 5.8 is available in the Epic Games Launcher today~~ | **Resolved r3:** Epic announced release on June 17, 2026, available on the Launcher — [announcement](https://www.unrealengine.com/news/unreal-engine-5-8-is-now-available) | — |
| Combination D actually compiles | Not installed | Install both, select Xcode 26.1.1 with `xcode-select` or `DEVELOPER_DIR`, rerun §3.1 and §3.2 |
| Xcode 16.4's toolchain works on macOS 26.6.2 (B) | Unsupported by Apple; not installed | Install it side by side, rerun §3.2 against UE 5.5.4 |
| Xcode 26.6 with UE 5.8 (E) — **compile: resolved r4, succeeds (§6a)**; **editor runtime: still unverified** | Editor not yet launched successfully | **Editor start resolved r5 (§6c).** Remaining: open a C++ project and play a level — the first Phoenix compile |
| UE 5.6 / 5.7 `MaxVersion` (C) | Not installed; forum inconclusive | Read `Apple_SDK.json` from either version |
| Editing `MaxVersion` is enough (F) | Not tried; newer clang can reject older engine source with errors the SDK check never reaches | Only in a disposable engine copy, never the shared install |
| macOS 26 above UE 5.8's *recommended* Sequoia 15 causes problems | Epic lists a minimum and a recommendation, no maximum | The §3.2 compile under D |

## 8. State left behind by this research

| what | where | status |
|---|---|---|
| Throwaway template copy and its build intermediates | session scratchpad `probe/` | disposable |
| UnrealBuildTool log | `~/Documents/Library/Application Support/Epic/UnrealBuildTool/Log.txt` | overwritten on every build |
| Engine install, Xcode, Phoenix repository | — | **unchanged** |

## Sources

- Epic — [macOS Development Requirements for Unreal Engine](https://dev.epicgames.com/documentation/unreal-engine/macos-development-requirements-for-unreal-engine?lang=en-US) (fetched 2026-09-14)
- Epic — [Using Modern Xcode in Unreal Engine](https://dev.epicgames.com/documentation/en-us/unreal-engine/using-modern-xcode-in-unreal-engine) (no version limits stated)
- Apple — [Xcode system requirements](https://developer.apple.com/xcode/system-requirements) (fetched 2026-09-14)
- Forum — [UE 5.6.1/5.7 not compiling on macOS 26.1 + Xcode 26.1.1](https://forums.unrealengine.com/t/unreal-engine-5-6-1-5-7-not-compiling-on-macos-26-1-xcode-26-1-1-platform-mac-errors/2675210)

## Related

- [[00-Research-Hub]] · [[Decisions]] · [[Git-LFS-Research-2026-09-14]]
