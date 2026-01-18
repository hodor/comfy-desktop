# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**ComfyUI Desktop** (@comfyorg/comfyui-electron) is an Electron-based desktop application that packages ComfyUI for end users. It handles Python environment setup, dependency management, and provides a seamless desktop experience for running AI diffusion models.

This is part of the Comfy Org ecosystem. Related repositories:
- [ComfyUI](https://github.com/comfyanonymous/ComfyUI) - Core Python backend
- [ComfyUI_frontend](https://github.com/Comfy-Org/ComfyUI_frontend) - Web frontend
- [ComfyUI-Manager](https://github.com/ltdrdata/ComfyUI-Manager) - Plugin manager

## Development Commands

```bash
# Code quality (run after changes)
yarn lint              # ESLint with auto-fix
yarn format            # Prettier formatting
yarn typescript        # Type checking

# Development
yarn start             # Build and launch with file watching
yarn make:assets       # Download ComfyUI dependencies (required before first run)

# Testing
yarn test:unit                    # Vitest unit tests
yarn test:unit path/to/file       # Run single test file
yarn test:e2e                     # Playwright E2E tests (requires COMFYUI_ENABLE_VOLATILE_TESTS=1)
yarn test:e2e:update              # Update E2E snapshots

# Building
yarn make              # Build platform package
yarn vite:compile      # Compile with Vite
```

### Troubleshooting

If you encounter `NODE_MODULE_VERSION` errors:
```bash
npx electron-rebuild
# If that fails:
yarn rebuild
```

## Architecture Overview

### Process Model

```
main.ts (entry point)
  └─ DesktopApp (orchestrator)
       ├─ AppWindow (window management)
       ├─ InstallationManager (setup/validation)
       └─ ComfyDesktopApp (runtime server management)
            └─ ComfyServer (Python process lifecycle)
```

### Key Architectural Patterns

**Two-Phase IPC Registration**: Handlers register at specific lifecycle stages:
- Phase 1 (before installation): path, network, app info, GPU handlers
- Phase 2 (after server starts): terminal, restart core handlers

**Service Locator Pattern**: Singletons with initialization guards:
- `useDesktopConfig()` - Electron-store wrapper for app settings
- `useComfySettings()` - Per-installation Comfy settings (YAML)
- `useAppState()` - Global state and event emitter

**AppState EventEmitter**: Loose coupling for lifecycle events:
- `ipcRegistered` - Fires once before IPC handler registration
- `loaded` - Fires when ComfyUI server is ready
- `installStageChanged` - Broadcasts installation progress to all windows

**Process Callback Pattern**: `createProcessCallbacks()` standardizes progress reporting from async operations (stdout/stderr streaming, progress updates via IPC).

### Source Structure

- `src/main.ts` - Entry point, app initialization
- `src/main-process/comfyDesktopApp.ts` - Server management, runtime IPC
- `src/main-process/comfyServer.ts` - Python process spawning, HTTP readiness polling
- `src/install/` - Installation wizard, validation, troubleshooting
- `src/handlers/` - IPC message handlers (organized by responsibility)
- `src/config/` - DesktopConfig (electron-store), ComfySettings (YAML)
- `src/preload.ts` - Context bridge (electronAPI exposed to renderer)

### Configuration Layers

1. **DesktopConfig** (`electron-store`): basePath, installState, selectedDevice, detectedGpu
2. **ComfySettings** (`basePath/config/comfy-settings.yaml`): Server.LaunchArgs, UV mirrors
3. **ComfyServerConfig** (`extra_models_config.yaml`): Model search paths

## Testing

### Unit Tests (Vitest)

- Location: `tests/unit/` (mirrors `src/` structure)
- Use `test.extend` over loose variables; import `test as baseTest` from vitest
- Use `test()` not `it()`
- Cross-platform: use `path.normalize`, `path.join`, `path.sep`
- Reference examples: `tests/unit/main-process/appState.test.ts`, `tests/unit/desktopApp.test.ts`

### E2E Tests (Playwright)

- Location: `tests/integration/`
- Requires: `COMFYUI_ENABLE_VOLATILE_TESTS=1` environment variable
- Always import from `testExtensions.ts`, not raw Playwright

**Test Projects** (run in order):
1. `install` - Fresh install tests
2. `post-install-setup` - Creates installed state
3. `post-install` - Tests requiring installed app
4. `post-install-teardown` - Cleanup

**Key Fixtures**:
```typescript
import { expect, test } from '../testExtensions';

test('example', async ({ app, window, installWizard, serverStart, troubleshooting }) => {
  // app.testEnvironment provides error simulation:
  await app.testEnvironment.breakInstallPath();
  await app.testEnvironment.breakVenv();
  // Auto-restored after test via Symbol.asyncDispose
});
```

## Code Style

### TypeScript Standards

- **No `any`**: Use `unknown` with type guards instead
- **ESM only**: No CommonJS (`require`, `module.exports`)
- **Explicit exports**: Exported functions need explicit parameter and return types
- **Immutability**: Prefer `readonly`, `as const`, `satisfies`
- **Error handling**: Throw `Error` subclasses, treat caught errors as `unknown`
- **Nullability**: Prefer `undefined` over `null`

### Documentation

Use JSDoc for exported APIs:
- Tags: `@param`, `@return` (omit for void), `{@link}` for symbol references
- Explain "why", not "what"

### Pre-Commit Checklist

1. `yarn format`
2. `yarn lint && yarn typescript`
3. `yarn test:unit`
4. Consider `yarn test:e2e` for UI changes

## Bundled Components

Versions defined in `package.json` under `config`:
- **ComfyUI**: Core backend (`config.comfyUI.version`)
- **ComfyUI_frontend**: Web UI (`config.frontend.version`)
- **uv**: Python package manager (`config.uvVersion`)

## Platform Paths

| Platform | App Data | Config File |
|----------|----------|-------------|
| Windows | `%APPDATA%\ComfyUI` | `config.json` |
| macOS | `~/Library/Application Support/ComfyUI` | `config.json` |
| Linux | `~/.config/ComfyUI` | `config.json` |

## Environment Variables

- `--dev-mode`: Force packaged app to read env vars
- `COMFY_HOST`/`COMFY_PORT`: Connect to external ComfyUI server
- `COMFYUI_ENABLE_VOLATILE_TESTS`: Enable E2E test execution
