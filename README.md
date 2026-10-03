# RobloxOS

RobloxOS is a desktop-style operating environment built inside a Roblox experience. It provides a sign-in and desktop flow, app windows and shortcuts, a taskbar, a command-line interface, a client-side kernel runtime, and an optional package catalog.


## Contents

- [At a glance](#at-a-glance)
- [System layout](#system-layout)
- [Desktop and included apps](#desktop-and-included-apps)
- [RoKernel and Quakewake](#rokernel-and-quakewake)
- [RoKernel CLI](#rokernel-cli)
- [Building an application](#building-an-application)
- [Building an RTML package](#building-an-rtml-package)
- [Available packages](#available-packages)
- [Persistence and boot configuration](#persistence-and-boot-configuration)
- [Security notes](#security-notes)
- [Testing checklist](#testing-checklist)
- [DataModel reference](#datamodel-reference)

## At a glance

| Area | Implementation |
| --- | --- |
| Desktop UI | `StarterGui.Desktop`, cloned into the player's `PlayerGui` |
| Kernel runtime | `ReplicatedStorage.RoKernel.RoKernel` |
| Window manager | `ReplicatedStorage.RoKernel.Quakewake` |
| Package manager | `ReplicatedStorage.RoKernel.PackageManager` and `upacman` CLI |
| Package API layer | RTML services in `ReplicatedStorage.RoKernel.RTMLServices` |
| Package catalog | `ServerStorage.RoKernelPackages.Catalog/<package-id>/ClientInstaller` |
| Server package and profile handler | `ServerScriptService.RoKernelPackageService` |
| Security configuration | `ServerScriptService.RoKernelSecurityConfig` |

The established place also contains a `robloxos-original` folder under `ServerStorage.RoKernelPackages`, separate from the installable `Catalog` packages.

## System layout

At a high level, startup works like this:

1. Roblox loads the boot and sign-in interface.
2. The player enters the desktop.
3. `StarterGui.Desktop.CanvasGroup.BackgroundApps.RoKernelBootstrap` waits for the desktop's `RoKernelBootTrigger` attribute to become `start`.
4. The bootstrap requires `ReplicatedStorage.RoKernel.RoKernel` and creates the client runtime.
5. RoKernel initializes Quakewake, the architecture policy, RTML services, and the package manager. The package manager loads the server catalog, restores installed packages, and applies the active boot configuration.

The runtime is client-side because it creates and manages the player's UI. Package definitions are catalogued on the server and copied to that player's `PlayerGui` when installed. Server code remains responsible for persistent data and privileged operations.

## Desktop and included apps

The desktop is under `StarterGui.Desktop.CanvasGroup`. Its main areas include:

- `Desktop`: desktop surface, right-click menu, and desktop shortcuts.
- `Apps`: built-in app definitions and their window templates.
- `ForegroundApps`: windows currently open on the desktop.
- `Taskbar`: running-app icons, app dock, clock, and date.
- `BackgroundApps`: system scripts for focus, taskbar behavior, screenshots, and kernel startup.
- `System.API`: shared desktop/window APIs.

The established place contains these built-in app entries:

`Task Viewer`, `Project Elite`, `UMPT`, `Settings`, `KekaExplorer`, `Speaker`, `Context`, `Command`, `PerformanceViewer`, `RoSearch`, `RenamePanel`, `PropertiesPanel`, and `RoKernelCLI`.

Some names describe the app's role directly; implementation and availability can vary by place version. The package catalog provides optional apps and tools described below.

## RoKernel and Quakewake

RoKernel is the client runtime and extension host. It initializes these main components:

- **Quakewake** registers apps, opens and closes windows, creates desktop shortcuts, applies themes, and arranges windows.
- **PackageManager** refreshes the server catalog, installs and removes packages, restores saved packages, manages boot configs, and provides a small virtual filesystem.
- **RTMLServices** validates declarative package API manifests and exposes only the services declared by the package.
- **Architecture** selects the app-routing and package-scanning policy.
- **Terminal** implements built-in and package-registered CLI commands.

Quakewake supports floating (`float`) and tiled (`tile`) window layouts. `Ctrl+Alt+Q` hides or restores open app windows. Supported themes are `classic`, `liquid-glass`, `graphite`, `terminal-green`, `amber`, and `linux-blue`.

Common commands:

```text
quakewake apps
quakewake layout tile
quakewake layout float
quakewake theme graphite
quakewake close-all
rokernel launch <app-id>
```

An app registered with Quakewake gets a window cloned from the desktop app template and a desktop shortcut. In architectures that require kernel routing, launch it through `rokernel launch <app-id>` rather than bypassing the kernel.

## RoKernel CLI

Open the **RoKernelCLI** app and enter commands there. Use `help` or `commands` to see the command catalog available in the running client. Package extensions can add commands, so the live catalog may differ from this list.

| Command | Purpose |
| --- | --- |
| `help` / `commands` | Show help and registered commands. |
| `info`, `whoami`, `date`, `time` | Show runtime or player information. |
| `upacman refresh` | Reload the server package catalog. |
| `upacman list` / `upacman installed` | List available or installed packages. |
| `upacman info <id>` | Show package metadata. |
| `upacman install <id>` / `upacman remove <id>` | Install or remove an optional package. |
| `rokernel architecture` | Show available architecture profiles. |
| `rokernel scan <id>` | Inspect a package's declared metadata. |
| `serverwall debug_report` | Report package scan findings and runtime state. |
| `quakewake layout tile\|float` | Change the window layout. |
| `quakewake theme <name>` | Change the desktop theme. |
| `ls`, `cat`, `write`, `mkdir` | Work with the RoKernel virtual filesystem. |
| `bootconfig list\|show\|save\|use\|delete` | Manage saved boot configurations. |
| `reset --makesave [name]` | Save the current package/theme/layout/filesystem profile, then reset optional packages and local state. |
| `reset --restore <name>` | Restore a saved system profile. |

The CLI's filesystem is virtual; these commands do not browse the Roblox DataModel or the player's computer. `sudo rm -rf <virtual-path>` is destructive within that virtual filesystem.

## Building an application

For an app that should be installable through `upacman`, create a folder and a `ClientInstaller` ModuleScript under:

```text
ServerStorage
  |-- RoKernelPackages
    |-- Catalog
      |-- hello-app
        `-- ClientInstaller (ModuleScript)
```

The installer returns package metadata and an `install(host)` hook. The hook registers an application through the declared `ApplicationService`. Quakewake creates the app window and shortcut; the `mount` function builds the app's UI inside the window.

```lua
local function mount(window, host, bootstrapArguments)
	local content = window.UsableArea:FindFirstChild("ScrollingFrame")
	if not content then
		return
	end

	local message = Instance.new("TextLabel")
	message.Name = "Message"
	message.Position = UDim2.fromOffset(12, 12)
	message.Size = UDim2.new(1, -24, 0, 32)
	message.BackgroundTransparency = 1
	message.Text = "Hello from RobloxOS"
	message.TextXAlignment = Enum.TextXAlignment.Left
	message.TextColor3 = Color3.fromRGB(235, 240, 242)
	message.Font = Enum.Font.Gotham
	message.TextSize = 16
	message.Parent = content

	local button = Instance.new("TextButton")
	button.Name = "ChangeMessage"
	button.Position = UDim2.fromOffset(12, 56)
	button.Size = UDim2.new(1, -24, 0, 36)
	button.Text = "Say hello"
	button.Parent = content
	button.Activated:Connect(function()
		message.Text = "Hello, " .. game.Players.LocalPlayer.DisplayName .. "!"
	end)
end

return {
	id = "hello-app",
	version = "1.0.0",
	description = "A small RobloxOS example application.",
	dependencies = {"quakewake"},
	permissions = {"ui"},
	apiConfig = [=[{"version":1,"services":[{"service":"ApplicationService","target":"RoKernel.Quakewake.AppRegistry","purpose":"Registers the Hello application.","config":{"appId":"HelloApp"}}]}]=],
	install = function(host)
		return host.Services.ApplicationService:registerApp({
			id = "HelloApp",
			title = "Hello",
			mount = mount,
		})
	end,
	uninstall = function(host)
		return host.Services.ApplicationService:unregisterApp("HelloApp")
	end,
}
```

After adding the ModuleScript in Studio, refresh the catalog and install it:

```text
upacman refresh
upacman info hello-app
upacman install hello-app
```

The installer `id`, catalog folder ID, and `ApplicationService` `appId` serve different purposes: the folder and returned `id` identify the package; the `appId` identifies the registered app. Keep IDs stable and unique. The app's `apiConfig` must be valid JSON and must match the server catalog metadata exactly.

### App lifecycle

1. The server discovers `Catalog/<id>/ClientInstaller` and exposes its metadata.
2. `upacman install <id>` asks the server to replicate the approved installer to the player's `PlayerGui`.
3. The client validates the installer and its API manifest, checks package dependencies, and calls `install(host)`.
4. `ApplicationService:registerApp()` registers a definition with Quakewake.
5. Quakewake creates the shortcut and window template. On launch it calls `mount(window, host, bootstrapArguments)`.
6. Removing the package calls `uninstall(host)` before removing its installed state. Unregister apps and other UI/resources in this hook.

For server-backed features, create a server-owned RemoteEvent/RemoteFunction and validate every client request on the server. Do not put secrets or authoritative game state in a LocalScript or package installer.

## Building an RTML package

RTML is the package API layer in this place. A package declares its requested services as JSON in `apiConfig`; JSON is configuration, not Luau code. Service IDs, targets, item counts, widget bounds, and action bindings are validated. Packages cannot use an RTML manifest to select arbitrary Instance paths.

Available services:

| Service | Fixed target | Use |
| --- | --- | --- |
| `TaskbarService` | `Desktop.CanvasGroup.Taskbar.AppContainer` | Register 1-12 taskbar items. `placement` is `center` or `left`. |
| `HomeService` | `Desktop.CanvasGroup.Desktop.DesktopShortcuts` | Register 1-24 desktop shortcuts. |
| `ApplicationService` | `RoKernel.Quakewake.AppRegistry` | Register one app whose ID matches the manifest's `appId`. |
| `ClientService` | `RoKernel.ClientRuntime` | Apply a declared theme using `setTheme()` and restore it on removal. |
| `WidgetService` | `Desktop.CanvasGroup.Desktop.RTMLWidgetLayer` | Mount desktop widgets or taskbar popovers. |
| `TerminalService` | `RoKernel.Terminal.Commands` | Register 1-24 CLI commands. |

Every service entry needs `service`, the exact `target`, a short `purpose`, and service-specific `config`. The full manifest is limited to six services and 32,000 bytes. Taskbar/home action IDs, widget mount IDs, and widget action IDs must be bound to functions returned in the package definition (`actions` or `widgetMounts`). Register them during `install`; unregister them during `uninstall`.

For a taskbar launcher, the manifest config looks like this:

```json
{
  "version": 1,
  "services": [
    {
      "service": "TaskbarService",
      "target": "Desktop.CanvasGroup.Taskbar.AppContainer",
      "purpose": "Adds a launcher for the package app.",
      "config": {
        "placement": "center",
        "items": [
          { "id": "hello", "label": "Hello", "action": "openHello" }
        ]
      }
    }
  ]
}
```

The package definition must bind `actions.openHello` to a function. That callback receives `(host, itemConfig, button)`. `TaskbarService:register()` and `:unregister()` are available through `host.Services.TaskbarService` while a package lifecycle hook is running.

## Available packages

These package IDs were present in the established place's server catalog when inspected:

| Package ID | Description |
| --- | --- |
| `assistant` | RobloxOS Assistant package with a taskbar launcher and popover. It can answer questions and prepare command proposals for user review. |
| `calculator` | Basic four-operation calculator app. |
| `calendar` | Calendar utility. |
| `clock` | Clock utility. |
| `credits` | Project credits. |
| `liquid-glass` | Liquid-glass appearance package. |
| `luau-executor` | Luau execution tool; treat as high risk. |
| `notepad` | Notepad package. |
| `notes` | Lightweight session notes app. |
| `quakewake-manager` | Quakewake management package; the runtime attempts to install it if it is in the catalog and is not already installed. |
| `system-monitor` | System monitor utility. |
| `terminal-tools` | Adds Unix-style commands for the RoKernel virtual filesystem. |
| `theme-amber` | Amber theme package. |
| `theme-graphite` | Graphite theme package. |
| `theme-linux-blue` | Linux-blue theme package. |
| `theme-terminal-green` | Terminal-green theme package. |

Install only packages you trust. Use `upacman info <id>` and `rokernel scan <id>` to inspect the catalog metadata available to the client before installing.

## Persistence and boot configuration


# The datastore argument is pretty much depreciated since i realised that it DOES boot but however won't recover past data without datastore so the --no-datastore bootargument is pretty much useless

`RoKernelPackageService` stores per-player installed-package state, boot configs, and saved system profiles in the `RoKernelProfilesV1` DataStore when DataStore access is available. Studio boot arguments can disable DataStore use with `--no-datastore` or `datastore=false`; in that case relevant changes are session-only. The CLI's `bootargs` command reports the runtime's view of these settings.

System profiles save installed optional packages, theme, layout, and virtual filesystem entries. For example:

```text
reset --makesave before-changes
reset --list
reset --restore before-changes
```

Boot configs can assign arguments to a package or run a virtual filesystem file during startup. A line has this form:

```text
/pkg/hello-app --bootstrap config args[]: --greeting "Hello"
```

Other way to perform boot arguments are through Command Code:
```lua
local storage = game:GetService("ServerStorage")
local args = storage:FindFirstChild("RoKernelBootArguments") or Instance.new("StringValue")
args.Name = "RoKernelBootArguments"
args.Value = "--no-datastore"
args.Parent = storage
```

for right now there is only one boot argument, and thats `no datastore`.


The package receives its arguments as the third argument to `mount(window, host, bootstrapArguments)`. Save the virtual file as a boot config with `bootconfig save <name> <virtual-file>`, then activate it using `bootconfig use <name>`. Boot config files that contain Luau are compiled and executed at startup; only use code you wrote or reviewed.

## Security notes

The established place includes package metadata scanning, architecture selection, a server-side package catalog, and a script interception service. These are defense-in-depth features, not a security sandbox. Roblox clients are not trusted: a modified client can bypass client-side checks, and the runtime cannot inspect arbitrary ModuleScript source in all cases.

The architecture module recognizes:

- `serverwall-1`: kernel-routed apps with heuristic package warnings.
- `serverwall-2-security`: kernel-routed apps; packages with high-risk or unverified findings are blocked by its scanner.
- `andrenaline`: direct app execution without firewall enforcement.

The default architecture value is `serverwall-1` when no recognized value is configured. The current server security config enables Luau execution, enables the UDMM2 enforcement setting, allows security-check overrides, and allows arbitrary/vhome execution paths. Check `serverwall --security_status` and server-side configuration for the live state; do not assume a client command alone proves server enforcement.

Permission manifests such as `ui`, `local-files`, `dynamic-code`, `external-http`, `persistent-data`, or `remote-control` are metadata used for review/scanning. They do not grant or sandbox Roblox engine capabilities. Keep privileged work on the server, validate and rate-limit remote requests, and never trust client-reported permissions or scan results. Avoid the Luau executor, override switches, arbitrary boot scripts, numeric asset requires, and externally sourced code in production experiences unless you have deliberately reviewed the risk.

The Assistant uses Roblox's server-side `TextGenerator` when available, filters player prompts and displayed responses through `TextService`, and proposes at most one command at a time. A proposal must be reviewed and explicitly confirmed before execution. The Command Luau Prompt is arbitrary client-side code and is higher risk than a RoKernel CLI command.

## Testing checklist

- Test in the **RobloxOS-Established** place, not only in the separate development environment.
- Verify sign-in, desktop boot, taskbar, desktop shortcuts, and app open/close behavior.
- Test an app in floating and tiled layouts; test window focus and `Ctrl+Alt+Q`.
- For packages, test catalog refresh, install, launch, uninstall, dependency errors, and reinstall after a fresh join.
- Test with DataStore access enabled and disabled; confirm the UI reports session-only state when persistence is unavailable.
- Validate each `apiConfig` as JSON and confirm it matches the server package definition exactly.
- Test server-backed features with invalid, repeated, and unauthorized remote requests.
- Check both the client and server output for errors and warnings.

## DataModel reference

```text
ReplicatedFirst
  SetupBoot

ReplicatedStorage
  RoKernel
    RoKernel
    Quakewake
    PackageManager
    Terminal
    Architecture
    RTMLServices
    PackageUi
    PackageRequest / AssistantRequest / NotepadSync / SecurityQuery

ServerScriptService
  RoKernelPackageService
  RoKernelAssistantService
  RoKernelSecurityConfig
  RoKernelSecurityService
  RoKernelSecurityDatabase
  RoKernelInterceptService
  NotepadDataService

ServerStorage
  |-- RoKernelPackages
      |-- Catalog/<package-id>/ClientInstaller
      `-- robloxos-original

StarterGui
  Desktop
    CanvasGroup
      Apps
      BackgroundApps
      Desktop
      ForegroundApps
      System
      Taskbar
```

For the in-experience package API reference, see the documentation ModuleScript at `ReplicatedStorage.RoKernel.README`. That module returns Markdown text; it is not an installer.
