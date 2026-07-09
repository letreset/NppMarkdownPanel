# AGENTS.md

Notepad++ plugin that previews Markdown. It is a **C# .NET Framework 4.7.2 WinForms class library** exported as a native Notepad++ plugin via UnmanagedExports. Windows-only; builds with MSBuild/Visual Studio (no `dotnet` CLI).

## Build

- `./build.ps1` is the canonical build. It locates MSBuild via `vswhere` and builds **Release x86 AND x64** (a release needs both). CI (`.github/workflows/CI_build.yml`) runs `build.ps1` then `makerelease.ps1`.
- Only the `NppMarkdownPanel` project is platform-specific (x86/x64, distinct output dirs `bin\Debug`, `bin\Debug-x64`, `bin\Release`, `bin\Release-x64`). The other 3 projects are AnyCPU.
- Restore uses `packages.config` (`/restore /p:RestorePackagesConfig=true`), not PackageReference. The `packages/UnmanagedExports.Repack.Upgrade.*/build/` targets folder is committed (`.gitignore` keeps `packages/build/`) and imported by the csproj — do not delete it.
- **C# 7.3 only** in the main project (`<LangVersion>7.3</LangVersion>`); don't use newer language features there.

## Testing

- **No automated test project.** Verification is manual: build, deploy, then open the sample docs in `NppMarkdownPanel/Resources/nppMdP.tests/*.md` inside Notepad++.
- Deploy Debug build with `copy-debug-x64.cmd` (paths are hard-coded to this machine — adjust before use). It copies the main DLL to `plugins\NppMarkdownPanel\` and the helper DLLs to `plugins\NppMarkdownPanel\lib\`.
- Debugging (F5) launches the installed `notepad++.exe` (`StartProgram` in the csproj, chosen by platform).

## Architecture

Four projects (see `NppMarkdownPanel.sln`):

- **NppMarkdownPanel** — the plugin DLL. Entry point `Main.cs` (UnmanagedExports surface); orchestration in `MarkdownPanelController.cs`; WinForms UI in `Forms/`; Notepad++/Scintilla P/Invoke interop in `PluginInfrastructure/`; IE11 renderer in `Webbrowser/`.
- **PanelCommon** — shared interfaces (`IWebbrowserControl`, `IMarkdownGenerator`) implemented across projects.
- **MarkdigWrapper** — Markdig-based Markdown→HTML plus syntax highlighting.
- **Webview2Viewer** — WebView2 (Edge/Chromium) renderer; the default engine (IE11 is the legacy fallback).

**Runtime assembly loading (key gotcha):** the 3 helper projects are `ProjectReference` with `Private=False`, so their DLLs are NOT copied next to the main DLL. At runtime `MarkdownPanelController.CurrentDomain_AssemblyResolve` (`MarkdownPanelController.cs:75`) resolves them from a `lib\` subfolder. **Adding any new runtime dependency requires adding a copy step to `makerelease.ps1`** (and the deploy `.cmd`) so it lands in `lib\`.

## Conventions & notes

- Settings persist in the Notepad++ plugin config dir as `NppMarkdownPanel.ini` (`SetIniFilePath` / `LoadSettingsFromIni`), read via Win32 INI APIs — not just the Settings dialog. New settings need a matching `ReadIniValue`/`ReadIniBool` line plus a `Settings` field.
- `NppMarkdownPanel` uses unsafe blocks and a COM reference to `SHDocVw` (IE11); keep those references when editing the csproj.
- CSS themes `style.css` / `style-dark.css` are copied to output and shipped in the release zip.
- `makerelease.ps1` names release zips from `NppMarkdownPanel.dll`'s FileVersion (set in `Properties/AssemblyInfo.cs`).
