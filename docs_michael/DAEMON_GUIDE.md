# OpenClaw Daemon System Guide

## Overview

The `src/daemon` directory implements cross-platform daemon/service management for OpenClaw's gateway and node processes. It provides a unified abstraction layer over platform-specific service managers:

- **macOS**: LaunchAgents (launchd)
- **Linux**: systemd user services
- **Windows**: Scheduled Tasks (schtasks)

This system enables OpenClaw to run as a background service that:
- Starts automatically on system boot or user login
- Restarts automatically on crashes
- Runs independently of terminal sessions
- Maintains proper environment variables and working directories
- Logs output to dedicated files

## Architecture

### Core Abstraction: `GatewayService`

The system is built around a platform-agnostic `GatewayService` interface defined in `service.ts`:

```typescript
export type GatewayService = {
  label: string;                    // Platform-specific name (e.g., "LaunchAgent", "systemd")
  loadedText: string;                // Status text when loaded
  notLoadedText: string;             // Status text when not loaded
  stage: (args) => Promise<void>;    // Write config without activating
  install: (args) => Promise<void>;  // Write config and activate service
  uninstall: (args) => Promise<void>;// Remove service
  stop: (args) => Promise<void>;     // Stop running service
  restart: (args) => Promise<void>;  // Restart service
  isLoaded: (args) => Promise<boolean>; // Check if service is active
  readCommand: (env) => Promise<GatewayServiceCommandConfig | null>; // Read config
  readRuntime: (env) => Promise<GatewayServiceRuntime>; // Read runtime status
};
```

Platform-specific implementations are registered in `GATEWAY_SERVICE_REGISTRY` and resolved via `resolveGatewayService()`.

### Service Lifecycle

1. **Stage**: Write service configuration files without activation
   - Useful for testing configurations before deployment
   - Files written but service not started

2. **Install**: Write configuration and activate service
   - Creates service definition files
   - Enables service to start on boot/login
   - Starts the service immediately
   - Platform-specific: backs up existing configs

3. **Start/Restart**: (Re)start the service
   - Clears "disabled" state (important for manual stops)
   - Handles in-process restarts (detached handoff)
   - Cleans up stale processes before restart

4. **Stop**: Stop service but keep installed
   - Disables auto-restart behavior
   - Preserves configuration for future starts

5. **Uninstall**: Remove service completely
   - Stops the service
   - Removes configuration files
   - On macOS: moves plist to Trash

### Service State Model

Services have a multi-faceted state tracked in `GatewayServiceState`:

```typescript
export type GatewayServiceState = {
  installed: boolean;  // Config file exists
  loaded: boolean;     // Service registered with system manager
  running: boolean;    // Process is currently executing
  env: GatewayServiceEnv;
  command: GatewayServiceCommandConfig | null;
  runtime?: GatewayServiceRuntime;
};
```

## Platform-Specific Implementations

### macOS: LaunchAgents (`launchd.ts`)

**Service Definition**: XML plist files in `~/Library/LaunchAgents/`
- Label format: `ai.openclaw.gateway` (or `ai.openclaw.<profile>` for named profiles)
- Logs: `~/.openclaw/gateway-*.log`

**Key Features**:
- GUI session requirement: LaunchAgents only work when user is logged into macOS GUI
- SSH/headless contexts fail with explicit error messages
- Restart handoff: When restarting from within the service process, uses detached handoff to avoid SIGTERM during restart
- `disable` state: Persists across bootout/plist changes, must be explicitly cleared

**Commands**:
- `launchctl bootstrap gui/<uid> <plist>` - Register and start
- `launchctl enable <target>` - Clear disabled state
- `launchctl kickstart -k <target>` - Restart service
- `launchctl disable <target>` - Prevent auto-restart
- `launchctl bootout <domain> <plist>` - Unregister
- `launchctl print <target>` - Get runtime status

**Special Handling**:
- Legacy label cleanup: Removes old service labels during install
- Secure directory creation: Sets proper permissions on LaunchAgents dir
- State probing: Waits for confirmation after stop to ensure clean shutdown

### Linux: systemd (`systemd.ts`)

**Service Definition**: Unit files in `~/.config/systemd/user/`
- Unit name format: `openclaw-gateway.service` (or `openclaw-gateway-<profile>.service`)
- Supports EnvironmentFile for loading vars from state dir

**Key Features**:
- User scope vs machine scope: Handles both `--user` and `--machine <user>@` for sudo contexts
- Linger support: Can enable user services to run without login session
- Environment files: Supports loading vars from `~/.openclaw/gateway.systemd.env`
- Backup on overwrite: Saves existing unit to `.bak` before replacing

**Commands**:
- `systemctl --user daemon-reload` - Reload service definitions
- `systemctl --user enable <unit>` - Enable auto-start
- `systemctl --user restart <unit>` - Restart service
- `systemctl --user disable --now <unit>` - Disable and stop
- `systemctl --user is-enabled <unit>` - Check enabled status
- `systemctl --user show <unit>` - Get runtime properties

**Special Handling**:
- Unavailable detection: Checks for missing systemd, bus unavailable, etc.
- Legacy unit cleanup: Removes old service names (e.g., `clawdbot-gateway`)
- Multiline env vars: Rejects multiline values in EnvironmentFile (systemd limitation)

### Windows: Scheduled Tasks (`schtasks.ts`)

**Service Definition**: CMD script in state dir + Scheduled Task registration
- Task name format: `OpenClaw Gateway` (or `OpenClaw Gateway (<profile>)`)
- Script location: `~/.openclaw/gateway.cmd`
- Startup fallback: Can also use `%APPDATA%\...\Startup\` for simpler setup

**Key Features**:
- Two-layer architecture: CMD script + scheduled task
- Access denied fallback: If schtasks fails with access denied, falls back to Startup folder
- Process tree cleanup: Kills entire process tree when stopping (Windows doesn't have proper signal handling)
- Port-based verification: Uses port listening to verify gateway is actually running

**Commands**:
- `schtasks /Create /TN <name> /TR <script> /SC ONLOGON` - Create task
- `schtasks /Run /TN <name>` - Start task
- `schtasks /End /TN <name>` - Stop task (then kills process tree)
- `schtasks /Delete /TN <name>` - Remove task
- `schtasks /Query /TN <name>` - Get status

**Special Handling**:
- Quoting hell: Separate quoting strategies for schtasks (`/TR`) vs cmd.exe script
- User resolution: Resolves `DOMAIN\username` format for task user
- Timeout handling: Has timeout detection and fallback mechanisms
- Filename sanitization: Removes invalid characters for Windows filenames

## Key Modules

### `service.ts` - Main Service Abstraction
- Exports unified `GatewayService` interface
- Platform registry and resolver
- High-level operations: `readGatewayServiceState()`, `startGatewayService()`
- Outcome descriptions for user feedback

### `service-types.ts` - Type Definitions
Core types for service configuration:
- `GatewayServiceInstallArgs`: Program args, working dir, environment, description
- `GatewayServiceState`: Multi-faceted service state
- `GatewayServiceCommandConfig`: Parsed command configuration from service files
- `GatewayServiceRuntime`: Runtime status (PID, state, exit codes)

### `constants.ts` - Service Naming
- Service label/name resolution for each platform
- Profile suffix handling (e.g., `--profile dev` → `ai.openclaw.dev`)
- Legacy label management
- Service description formatting

### `gateway-entrypoint.ts` - Entrypoint Resolution
Finds the correct JavaScript entrypoint for the gateway:
- Searches for `dist/index.js`, `dist/entry.js`, etc.
- Validates paths match expected pattern
- Used during install to find what to run

### `service-env.ts` - Environment Building
Constructs proper environment for services:
- **PATH construction**: Includes npm global, nvm, fnm, volta, pnpm, bun, asdf
- **Platform differences**: macOS vs Linux bin directories (e.g., `~/Library/pnpm` vs `~/.local/share/pnpm`)
- **Proxy support**: Preserves HTTP_PROXY, HTTPS_PROXY, NO_PROXY
- **TLS certificates**: Injects NODE_EXTRA_CA_CERTS for macOS LaunchAgents and nvm-installed Node
- **Service metadata**: Adds OPENCLAW_SERVICE_MARKER, OPENCLAW_SERVICE_KIND, version

### `runtime-binary.ts` - Runtime Detection
Identifies Node.js vs Bun from executable path:
- Handles versioned node binaries (`node-18`, `node22.3.0`)
- Recognizes `bun` and `bun.exe`

### `diagnostics.ts` - Error Detection
Reads gateway logs to detect common errors:
- Port binding failures
- Auth mode issues
- Gateway start blocks
- Tailscale requirements

### `launchd-restart-handoff.ts` - macOS Restart Coordination
Handles the tricky case of restarting from within the managed process:
- Spawns detached helper process
- Helper waits for parent PID to exit
- Helper then issues `launchctl kickstart -k`
- Prevents SIGTERM race during restart

### `exec-file.ts` - Command Execution
UTF-8 wrapper around Node's `execFile`:
- Ensures stdout/stderr are UTF-8 decoded
- Returns structured result with code, stdout, stderr
- Used by all platform-specific implementations

### `output.ts` - Terminal Formatting
Consistent output formatting:
- `formatLine(label, value)`: Aligned key-value output
- `writeFormattedLines()`: Batch output with optional blank lines
- Path normalization for cross-platform display

### `restart-logs.ts` - Log Path Resolution
Determines where service logs are written:
- macOS: `~/.openclaw/gateway-stdout.log`, `~/.openclaw/gateway-stderr.log`
- Linux: journalctl (systemd) or similar paths
- Windows: Script redirects to log files

## Configuration and Environment

### Environment Variables

**Service Configuration**:
- `OPENCLAW_PROFILE`: Service profile (affects service name/label)
- `OPENCLAW_GATEWAY_PORT`: Gateway listening port
- `OPENCLAW_STATE_DIR`: State directory path
- `OPENCLAW_CONFIG_PATH`: Config file path

**Platform Overrides**:
- `OPENCLAW_LAUNCHD_LABEL`: Override macOS LaunchAgent label
- `OPENCLAW_SYSTEMD_UNIT`: Override Linux systemd unit name
- `OPENCLAW_WINDOWS_TASK_NAME`: Override Windows task name
- `OPENCLAW_TASK_SCRIPT`: Override Windows script path

**Runtime Markers**:
- `OPENCLAW_SERVICE_MARKER`: Always `"openclaw"` for gateway services
- `OPENCLAW_SERVICE_KIND`: `"gateway"` or `"node"` depending on service type
- `OPENCLAW_SERVICE_VERSION`: OpenClaw version string

**PATH Enhancement**:
Services get minimal but comprehensive PATH including:
- System bins: `/usr/local/bin`, `/usr/bin`, `/bin`
- User bins: `~/.local/bin`, `~/bin`
- npm global: `~/.npm-global/bin`
- Version managers: nvm, fnm, volta, asdf
- Package managers: pnpm, bun

### Program Arguments

Services are configured with full program arguments:
- Runtime: `node` or `bun` (resolved from current process)
- Entrypoint: Resolved gateway dist file
- Flags: `--port`, `--profile`, etc.
- Working directory: Optional CWD for the service

Example:
```
["/usr/local/bin/node", "/path/to/openclaw/dist/index.js", "--port", "3000"]
```

## Important Patterns and Conventions

### Error Handling

**Graceful Degradation**:
- Service unavailable → Clear error messages
- Config read failures → Return null, don't throw
- Runtime status errors → Return "unknown" status

**Platform Detection**:
- Check for command availability before executing
- Detect missing systemd, unavailable bus, etc.
- Provide actionable error messages (e.g., "sign in to macOS GUI")

### Testing Considerations

**Test Structure**:
- Unit tests: `*.test.ts` - Pure logic, mocked exec
- Integration tests: `*.integration.e2e.test.ts` - Real system calls
- Test helpers: `service.test-helpers.ts`, `test-helpers/`

**Mocking**:
- Mock `exec-file.ts` for unit tests
- Mock filesystem for config parsing tests
- Use real platform commands for e2e tests

**Platform-Specific**:
- Skip tests on wrong platform: `test.skipIf(process.platform !== 'darwin')`
- Use fixtures for realistic command output
- Test both success and error paths

### Service State Transitions

```
Not Installed
    ↓ install()
Installed + Loaded + Running
    ↓ stop()
Installed + Loaded + Stopped
    ↓ restart()
Installed + Loaded + Running
    ↓ uninstall()
Not Installed
```

**Special States**:
- **Cached Label** (macOS): Service registered but plist deleted
- **Missing Unit** (Linux): Service queried but unit file not found
- **Disabled** (macOS): Service exists but won't auto-start
- **Startup Fallback** (Windows): Task registration failed, using Startup folder

### Restart from Within Service

**Problem**: If gateway calls `restart()` on itself, `kickstart -k` sends SIGTERM before completion.

**Solution** (macOS):
1. Detect if current process is the managed service (check `XPC_SERVICE_NAME`)
2. Spawn detached helper process with `scheduleDetachedLaunchdRestartHandoff()`
3. Helper waits for parent PID to exit
4. Helper issues `launchctl kickstart -k`
5. Original process returns "scheduled" outcome

### PATH Construction Strategy

**Goal**: Services should find `node`, `npm`, `pnpm`, etc. without inheriting full shell env.

**Approach**:
1. Include env-configured paths first (e.g., `PNPM_HOME`, `NVM_DIR`)
2. Add common user bin directories
3. Add platform-specific version manager paths
4. Add system paths last
5. Deduplicate entries

**Platform Differences**:
- macOS: `~/Library/Application Support/fnm`, `~/Library/pnpm`
- Linux: `~/.local/share/fnm`, `~/.local/share/pnpm`

### Quote Escaping Hell (Windows)

**Problem**: Windows has multiple parsing layers:
1. PowerShell/CMD command line parsing
2. `schtasks.exe` parsing of `/TR` argument
3. CMD.exe parsing of generated script

**Solution**:
- Use separate quoting functions: `quoteSchtasksArg()` vs `quoteCmdScriptArg()`
- Test with spaces, quotes, and special characters
- Validate round-trip parsing

## Common Operations

### Install Gateway Service

```typescript
import { resolveGatewayService } from './src/daemon/service.js';

const service = resolveGatewayService();
await service.install({
  env: process.env,
  stdout: process.stdout,
  programArguments: ['node', '/path/to/dist/index.js', '--port', '3000'],
  workingDirectory: '/path/to/openclaw',
  environment: { OPENCLAW_PROFILE: 'dev' },
  description: 'OpenClaw Gateway (dev, v2024.1.1)',
});
```

### Check Service Status

```typescript
import { readGatewayServiceState, resolveGatewayService } from './src/daemon/service.js';

const service = resolveGatewayService();
const state = await readGatewayServiceState(service);

console.log('Installed:', state.installed);
console.log('Loaded:', state.loaded);
console.log('Running:', state.running);
console.log('PID:', state.runtime?.pid);
```

### Restart Service

```typescript
const service = resolveGatewayService();
const result = await service.restart({
  stdout: process.stdout,
  env: process.env,
});

if (result.outcome === 'scheduled') {
  console.log('Restart scheduled (in-process handoff)');
} else {
  console.log('Service restarted');
}
```

## Troubleshooting

### macOS: "LaunchAgent install requires a logged-in macOS GUI session"

**Cause**: Running from SSH, headless context, or as wrong user.

**Fix**: Sign in to macOS GUI as the target user and run install command from Terminal.app.

### Linux: "systemctl --user unavailable"

**Causes**:
- systemd not installed
- User D-Bus session not available
- Running from cron or non-login shell

**Fixes**:
- Install systemd
- Enable linger: `loginctl enable-linger`
- Set `XDG_RUNTIME_DIR=/run/user/$(id -u)`

### Windows: "Access is denied" when creating scheduled task

**Causes**:
- Insufficient permissions
- Group Policy restrictions
- Scheduled Tasks service disabled

**Fixes**:
- Run as Administrator
- Check Group Policy: `gpedit.msc` → Computer Configuration → Administrative Templates → Windows Components → Task Scheduler
- Enable Task Scheduler service: `sc config Schedule start=auto`
- **Fallback**: System automatically tries Startup folder method

### Service Installed but Not Running

**Check**:
1. Read logs: `~/.openclaw/gateway-*.log` (macOS), `journalctl --user -u openclaw-gateway` (Linux), script log (Windows)
2. Check diagnostics: `readLastGatewayErrorLine()`
3. Verify PORT not in use
4. Check service state: `readGatewayServiceState()`

**Common Issues**:
- Port already bound
- Gateway auth mode mismatch
- Missing dependencies
- Wrong working directory

### Service Keeps Restarting

**Causes**:
- Crash loop (check exit status)
- Auto-restart policy too aggressive
- Port conflict with another instance

**Fixes**:
- Check `lastExitStatus` and `lastExitReason` in runtime state
- Review logs for crash reason
- Stop service: `service.stop()`
- Fix underlying issue before restarting

## Architecture Decisions

### Why Not pm2/supervisor/systemd-system?

**pm2/supervisor**: User-level process managers, but require Node.js to be running. We want OS-level supervision that works even if Node crashes or is unavailable.

**systemd system units**: Require root/sudo. User services (`--user`) allow per-user daemons without elevated privileges.

**Our approach**: Use OS-native service managers that:
- Require no additional dependencies
- Work without Node.js installed system-wide
- Support user-scoped services (no root needed)
- Integrate with OS logging and monitoring
- Auto-start on login/boot

### Why Separate Stage/Install?

**Stage**: Test configurations without activating.
- Useful for CI/CD: validate service files without starting services
- Allows review before activation
- Safe for multi-tenancy (shared systems)

**Install**: Activate immediately.
- Production deployments
- User expects service to start right away
- Enables auto-start on boot

### Why CMD Scripts on Windows?

**Alternatives Considered**:
1. Direct `schtasks /TR "node.exe ..."` - Escaping nightmare, env var expansion issues
2. PowerShell scripts - Execution policy restrictions, version compatibility
3. VBScript - Deprecated, limited, poor error handling

**CMD Scripts Win**:
- Universal (every Windows has cmd.exe)
- Simple variable expansion with `SET`
- Working directory control with `cd /d`
- Output redirection built-in
- Familiar escaping rules

### Why Not Docker/Containerization?

**Context**: OpenClaw targets desktop/workstation environments, not server deployments.

**Issues with Containers**:
- Desktop Docker requires user interaction (GUI, resource limits)
- No auto-start without additional setup
- Filesystem isolation complicates config/state
- Network isolation requires port mapping
- More complex for average users

**Our Target**: Native desktop app experience with background service.

## Future Considerations

### Multi-Instance Support

Currently: Profile-based instances share same service manager (different labels/names).

Future: Could support multiple gateways per profile with instance IDs.

### Remote Restart

Currently: Restart requires local shell access or API call to running gateway.

Future: Could support remote restart via gateway API with proper auth.

### Health Checks

Currently: Process existence + port listening.

Future: Could add HTTP health endpoint polling, structured health state.

### Log Rotation

Currently: Logs grow unbounded.

Future: Could add rotation by size/time, compression, retention policies.

### Windows Service (not Scheduled Task)

Currently: Uses Scheduled Tasks (easier, no admin required).

Future: Could offer true Windows Service option for enterprise deployments.

## Related Documentation

- Gateway Protocol: `src/gateway/protocol/`
- Process Management: `src/infra/restart-stale-pids.ts`
- Config System: `src/config/`
- Bootstrap: `src/bootstrap/`

## Summary

The daemon subsystem provides robust, cross-platform service management for OpenClaw. Key characteristics:

- **Unified API**: Single interface across macOS/Linux/Windows
- **OS-Native**: Uses platform service managers (no third-party deps)
- **User-Scoped**: No root/admin required for install
- **Auto-Start**: Services start on boot/login
- **Self-Healing**: Auto-restart on crashes
- **Environment Control**: Proper PATH, env vars, working directory
- **Graceful Errors**: Clear messages when platform features unavailable
- **Production Ready**: Handles edge cases, restart handoffs, legacy cleanup

When working with this code, remember:
1. Test on target platform (or use VM/CI)
2. Mock platform commands for unit tests
3. Handle "unavailable" gracefully (not all platforms support all features)
4. Preserve user customizations (backup existing configs)
5. Provide clear error messages with actionable fixes
