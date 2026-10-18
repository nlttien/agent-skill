---
name: single-file-packager-deployment
description: >-
  Building, packaging, self-extracting, 1-file JSON sync next to .exe, and releasing standalone All-In-One executables
  (AutoBoss_AllInOne.exe, AutoBuyer_AllInOne.exe) into dist/.
---

# Single-File Packager & Release Playbook

## 🔍 When to Use This Skill
- Packaging and publishing new bot releases to `dist/`.
- Fixing issues where new code changes are not reflected in the standalone `.exe`.
- Debugging `SingleFilePackager` payload extraction, caching, and locked files.
- 1-File JSON synchronization (`ShopAutoBuyer_settings.json` / `AutoExile_settings.json`) residing directly next to the standalone `.exe`.

---

## 🛠️ Step-by-Step Release Workflow

### 1. Build Standalone Boss Executable
```powershell
powershell -ExecutionPolicy Bypass -File "d:\codecuatien\ExileApi-Compiled\BUILD_BOSS_PACKAGE.ps1"
```
**Output**:
- `d:\codecuatien\ExileApi-Compiled\dist\AutoBoss_AllInOne.exe`
- `d:\codecuatien\ExileApi-Compiled\dist\AutoExile_settings.json` (1 file JSON duy nhất)

### 2. Build Standalone Buyer Executable
```powershell
powershell -ExecutionPolicy Bypass -File "d:\codecuatien\ExileApi-Compiled\BUILD_PACKAGE.ps1"
```
**Output**:
- `d:\codecuatien\ExileApi-Compiled\dist\AutoBuyer_AllInOne.exe`
- `d:\codecuatien\ExileApi-Compiled\dist\ShopAutoBuyer_settings.json` (1 file JSON duy nhất)

---

## 🛡️ Critical Packager Rules
1. **1-File JSON Synchronization Next to .EXE**:
   - `AutoBuyer_AllInOne.exe` detects `ShopAutoBuyer_settings.json` directly next to `.exe`.
   - `AutoBoss_AllInOne.exe` detects `AutoExile_settings.json` directly next to `.exe`.
   - When bot starts, it auto-loads that 1 JSON file.
   - When bot exits, it saves back to that 1 JSON file next to `.exe`.
2. **Always Overwrite Payload Files**:
   In `SingleFilePackager/Program.cs` and `SingleFilePackager_Boss/Program.cs`:
   ```csharp
   try { entry.ExtractToFile(destPath, true); } catch { /* Ignore locked files */ }
   ```
3. **Dual DLL Copy**:
   Always copy output assemblies to both `Plugins/Compiled/<Name>/` and `Plugins/Compiled/<Name>.dll`.
4. **Clean Temp Package**:
   Always remove stale `payload.zip` and `temp_pkg/` before zipping.

---

## 🐧 Đóng gói Linux cho app .NET (PoeRedux CLI) — 2026-10-17

- **Self-contained**: `dotnet publish -c Release -r linux-x64 --self-contained -p:DebugType=none` → máy đích không
  cần .NET. Native lib (vd `liboo2core.so`) khai báo `<None … CopyToPublishDirectory="PreserveNewest">`.
- **AppImage** (1 file, bấm đúp là chạy): AppDir = `usr/bin/` (bản publish) + `AppRun` (không tham số → mở GUI zenity,
  có tham số → chạy CLI) + `.desktop` + icon SVG (+ `.DirIcon`) → `ARCH=x86_64 appimagetool --no-appstream AppDir out.AppImage`
  (appimagetool 1.9.0). Máy đích cần FUSE (libfuse2) và quyền chạy (Properties → Is executable).
- Ổ `/` đầy: đặt .NET SDK, NUGET_PACKAGES, TMPDIR, DOTNET_CLI_HOME trên ổ khác (script env riêng).
- Release: `gh release create <tag> <file> -R <fork> --target <sha>`; tải về `cmp` với file gốc để xác nhận.
