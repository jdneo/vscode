# Computer-use integration baseline for browser-use

## Examined baseline and scope

This trace examines `microsoft/vscode` branch `research/browser-use-computer-use-baseline` at commit [`d0adb05e7252fd61c7a12d57681101a50b2d3de5`](https://github.com/microsoft/vscode/commit/d0adb05e7252fd61c7a12d57681101a50b2d3de5), committed on 2026-10-27 at 02:51:03 UTC. It follows the complete Computer Use path from configuration registration and Agent Host configuration mirroring through release resolution, cache publication, Copilot SDK startup, activation, MCP and operating-system permissions, renderer setup UI, restart/update behavior, and tests.

The evidence is limited to local source and public primary sources: VS Code source and history pinned to the examined commit, the official [`github/computer-use-plugin` v0.1.95 release](https://github.com/github/computer-use-plugin/releases/tag/v0.1.95) and [release API record](https://api.github.com/repos/github/computer-use-plugin/releases/tags/v0.1.95), and GitHub's pinned [Copilot SDK plugin-directory documentation](https://github.com/github/docs/blob/3e6eca1bee3d0abed640682488ffe8600ebd32b0/content/copilot/how-tos/copilot-sdk/features/plugin-directories.md#L191-L206). Relevant VS Code history starts with the [initial integration](https://github.com/microsoft/vscode/commit/5fdf1f59a19e137c1a6ae43bdeb2fae45a54fb1f), followed by [concurrent-install hardening](https://github.com/microsoft/vscode/commit/face4f0cb8cb4bec58031ac5b1fb11867e569ab2), [macOS setup support](https://github.com/microsoft/vscode/commit/66cbe1b8a8fe4f9d757d6eb01b341d21a94d4c6f), and [agent-driven activation](https://github.com/microsoft/vscode/commit/f2b5f7d31f4c75f410f200d7368ad8da4f05195a).

## Executive answer

Browser-use should reuse the integration's stable host and SDK seams: local-only configuration mirroring when the controlled browser is local; startup-configuration fingerprinting and idle-safe Copilot client restart; trusted `builtinPluginDirectories` injection; MCP provenance and controlling-setting metadata; defensive, versioned host/renderer wire readers; and the staged, hash-pinned, cancellation-aware publication pattern.

It should not copy the Computer Use implementation wholesale. The downloader, cache namespace, supported targets, helper-app discovery, macOS permission state machine, singleton plugin-directory wiring, machine-wide persistent activation, and runtime-recommended MCP auto-approval are Computer Use-specific assumptions. A second built-in plugin needs one authoritative descriptor/aggregation layer so both plugin directories coexist, an explicit browser locality and profile/domain consent model, and a deliberate activation scope. The existing update path is code-pin-driven rather than self-updating: a VS Code source change selects a new release, the new version installs into a new cache directory, and no polling, completed-cache revalidation, retry timer, or stale-version cleanup occurs.

## End-to-end lifecycle and data flow

1. **Register and mirror the gate.** `chat.agentHost.computerUse.enabled` is registered default-off and described as experimental/advanced. Its Agent Host metadata uses `AgentHostConfigurationSyncScope.Local`, so the value is mirrored only to the local utility-process Agent Host. Configuration forwarding sends root-configuration patches live, while workspace, folder, and memory values are not mirrored. Sources: [`agentHostStarter.config.contribution.ts`, setting registration](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/common/agentHostStarter.config.contribution.ts#L318-L325), [`configurationRegistry.ts`, Agent Host metadata](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/configuration/common/configurationRegistry.ts#L73-L83), and [`AgentHostProtocolClient._updateConfiguration`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/browser/agentHostProtocolClient.ts#L624-L650).
2. **Re-check locality in the owning process.** `CopilotAgent._isComputerUseEnabled` requires both the mirrored root value and `ILocalModelSetupService.isLocalAgentHost`; `CopilotComputerUsePlugin.isAvailable` then rejects unsupported platform/architecture targets. The gate is therefore enforced again where download and process startup occur, not trusted solely to configuration routing. Source: [`CopilotAgent._isComputerUseEnabled`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgent.ts#L1788-L1806).
3. **Resolve without blocking ordinary chat.** During shared Copilot client startup, `CopilotAgent._resolveComputerUsePlugin` returns a completed cached install immediately. Otherwise it starts a cancellation-aware background resolution, waits at most two minutes in the startup path, and allows Copilot to remain usable without Computer Use if resolution does not succeed. Sources: [startup call site](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgent.ts#L849-L854) and [`CopilotAgent._resolveComputerUsePlugin`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgent.ts#L1582-L1624).
4. **Download, verify, and publish atomically.** `CopilotComputerUsePlugin.download` selects the exact platform archive, requires an HTTP 2xx response, destroys the response body when cancellation interrupts a stalled transfer, buffers the archive, verifies its exact SHA-256, extracts only the target subtree into a staging directory, writes `.complete`, and publishes the staging directory with `rename`. Concurrent publication is treated as success only when the winning destination is complete. Sources: [`CopilotComputerUsePlugin.download`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotComputerUsePlugin.ts#L157-L218) and [extraction/publication path](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotComputerUsePlugin.ts#L220-L251).
5. **Inject the trusted directory at CLI construction.** The installed absolute path is passed to the SDK as `CopilotClientOptions.builtinPluginDirectories: [computerUsePluginPath]` before sessions are created. GitHub documents built-in plugin directories as host-supplied trusted plugins, distinct from user plugin discovery. Sources: [`CopilotAgent._createCopilotClient`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgent.ts#L4248-L4267) and [Copilot SDK built-in plugin directory contract](https://github.com/github/docs/blob/3e6eca1bee3d0abed640682488ffe8600ebd32b0/content/copilot/how-tos/copilot-sdk/features/plugin-directories.md#L191-L206).
6. **Offer activation only where it can work.** Durable sessions ask `CopilotComputerUseActivation.createTools()` for an eager `enable_computer_use` tool only while the trusted built-in plugin is present but disabled. Ephemeral sessions and scoped-credential sessions omit it. Sources: [`CopilotSessionLauncher` activation contribution](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotSessionLauncher.ts#L316-L320) and [durable-session tool assembly](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotSessionLauncher.ts#L1413-L1418).
7. **Collect explicit activation consent.** The activation tool uses `skipPermission: true` because the tool itself renders an explicit Turn On/Not Now card. It rejects managed-policy and unattended contexts, remembers a decline for the session, serializes concurrent requests, calls `session.rpc.plugins.enable`, and confirms the plugin is enabled before reporting success. Source: [`CopilotComputerUseActivation.createTools` and tool invocation](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotComputerUseActivationTool.ts#L125-L193).
8. **Route only trusted Computer Use permission elicitation.** On macOS, `CopilotAgentSession` enters the specialized setup path only when both `elicitationSource === 'computer-use'` and runtime provenance is `{ source: 'builtin', sourcePlugin: 'computer-use' }`. A same-named non-built-in source or malformed payload remains on the ordinary elicitation path. Source: [Computer Use elicitation routing in `CopilotAgentSession`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgentSession.ts#L8380-L8498).
9. **Coordinate operating-system setup without claiming to grant it.** `CopilotComputerUsePermissionSetup` validates versioned wire requests, serializes Accessibility and Screen Recording actions across chats, publishes host-verified transient status, and accepts only nonce/round-bound renderer responses. It answers the plugin's MCP elicitation; the plugin/helper and macOS remain authoritative for permission state. Sources: [`CopilotComputerUsePermissionSetup` request coordination](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotComputerUsePermissionSetup.ts#L292-L342) and [action/status handling](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotComputerUsePermissionSetup.ts#L592-L735).
10. **Restart the Copilot runtime when startup inputs change.** Download completion and relevant configuration changes request a Copilot client restart. `_requestClientRestart` waits until every turn and subagent is idle, stops the client, and lets the next operation recreate it with the updated startup inputs. Source: [`CopilotAgent` restart machinery](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgent.ts#L1977-L2108).

## Setting and local-host gate

The setting is intentionally an enablement gate, not the plugin's activation state. Setting `chat.agentHost.computerUse.enabled` permits the local Agent Host to resolve and supply the built-in plugin; the Copilot runtime can still report that plugin as disabled until the user accepts activation. This distinction supports default-off product rollout while retaining a separate end-user consent step. Sources: [setting registration](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/common/agentHostStarter.config.contribution.ts#L318-L325) and [`CopilotComputerUseActivation.createTools`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotComputerUseActivationTool.ts#L125-L193).

The local boundary is defense in depth:

- `AgentHostConfigurationSyncScope.Local` maps only to `AgentHostConfigurationSyncTarget.Local`, so the setting is not routed to remote Agent Hosts. Source: [`agentHostConfigurationSync.ts`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/common/agentHostConfigurationSync.ts#L109-L132).
- `CopilotAgent._isComputerUseEnabled` independently requires `isLocalAgentHost`. Source: [`CopilotAgent._isComputerUseEnabled`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgent.ts#L1788-L1806).
- The renderer advertises the specialized macOS setup only when `remoteAuthority` is absent, the provider is `copilotcli`, and the window is native macOS. Sources: [`AgentHostChatContribution` capability setup](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/workbench/contrib/chat/browser/agentSessions/agentHost/agentHostChatContribution.ts#L163-L165) and [local macOS predicate](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/workbench/contrib/chat/browser/agentSessions/agentHost/agentHostChatContribution.ts#L421-L424).

Browser-use should reuse this three-layer shape only if it controls a browser process and profile on the user's local machine. If browser execution can be remote, the descriptor must state where the browser lives and which host owns installation, credentials, profile data, UI consent, and cleanup rather than inheriting `Local` by analogy.

## Release pin

`COMPUTER_USE_PLUGIN_RELEASE` pins plugin version `0.1.95`, an exact GitHub release URL template, and an exact SHA-256 per supported target: `darwin-arm64`, `darwin-x64`, `win32-arm64`, and `win32-x64`. Linux is absent and therefore unavailable. Source: [`COMPUTER_USE_PLUGIN_RELEASE` and target table](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotComputerUsePlugin.ts#L22-L50).

The official v0.1.95 release was published on 2026-10-20 at 16:20:59 UTC, and its release API record exposes the four target assets and their digests. Primary sources: [official release](https://github.com/github/computer-use-plugin/releases/tag/v0.1.95) and [official release API](https://api.github.com/repos/github/computer-use-plugin/releases/tags/v0.1.95).

The Copilot SDK is pinned separately to `@github/copilot-sdk` version `1.0.27-unstable.40031404408.gba6c003`; the plugin release and SDK version therefore move independently and require compatibility discipline in VS Code updates. Source: [`package.json` dependency pin](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/package.json#L103-L108).

## Download and integrity verification

The resolver's useful reusable sequence is:

1. Derive a target from the actual Agent Host process platform and architecture.
2. Download only the source-pinned release URL and require an HTTP 2xx response.
3. Tie the response stream to cancellation and destroy it when cancellation interrupts a stalled body.
4. Compute SHA-256 over the downloaded archive and compare it to the target-specific source pin before extraction.
5. Extract only the expected target subtree into a unique staging directory.
6. Write `.complete` after extraction succeeds.
7. Rename staging to the versioned destination, tolerating a concurrent winner only if the published destination is complete.
8. Clean incomplete staging/destination state on failure.

The owning implementation is [`CopilotComputerUsePlugin.download`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotComputerUsePlugin.ts#L157-L218) plus its [extraction and atomic publication logic](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotComputerUsePlugin.ts#L220-L251).

VS Code deliberately excludes the Copilot SDK platform package's bundled Computer Use payload from product packaging because product signing would re-sign it; the separate GitHub release download preserves the upstream signature. Sources: [`build/lib/copilot.ts` packaging exclusion](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/build/lib/copilot.ts#L140-L155) and [`copilotComputerUsePlugin.ts` signing rationale](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotComputerUsePlugin.ts#L30-L40).

The integrity boundary is strong before first publication but weak afterward: `.complete` becomes the sole cache-validity check. Startup does not re-hash installed files, validate `plugin.json`, or repair a tampered or partially corrupted directory that retained the sentinel. Source: [`CopilotComputerUsePlugin.getInstalledPath`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotComputerUsePlugin.ts#L141-L151).

## Cache layout

The plugin cache is:

```text
<userDataPath>/
  agent-host/
    computer-use-plugin/
      0.1.95/
        <platform>-<arch>/
          .complete
          ...extracted plugin payload...
```

The version and target are both part of the destination, so a source pin bump naturally installs side by side rather than mutating the old payload. Source: [`CopilotComputerUsePlugin` cache path construction](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotComputerUsePlugin.ts#L120-L151).

The macOS helper application is not read from that cache. Its path is rooted at `COPILOT_COMPUTER_USE_HOME` or `COPILOT_HOME`, resolves relative paths against the user home, and appends `computer-use/Copilot Computer Use.app`. This is a separate shared Copilot helper-home contract and should not become a generic browser-use cache convention. Source: [`resolveComputerUseHelperAppPath`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotComputerUsePlugin.ts#L55-L68).

## CLI SDK injection

The stable SDK seam is `CopilotClientOptions.builtinPluginDirectories`, an ordered array of absolute paths provided when the shared Copilot client is constructed. GitHub's SDK documentation identifies these as trusted host-provided plugins, which means VS Code, not ordinary plugin discovery, owns provenance and installation. Sources: [`CopilotAgent._createCopilotClient`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgent.ts#L4248-L4267) and [SDK documentation](https://github.com/github/docs/blob/3e6eca1bee3d0abed640682488ffe8600ebd32b0/content/copilot/how-tos/copilot-sdk/features/plugin-directories.md#L191-L206).

The current implementation constructs a singleton array containing only `computerUsePluginPath`. Browser-use cannot safely add a second independent assignment; one authoritative startup step must aggregate every resolved built-in plugin path in deterministic order and include the resulting descriptor set in the startup fingerprint. Otherwise one integration can overwrite or omit the other.

Scoped-credential Copilot clients deliberately omit the built-in directory, and sessions using scoped GitHub credentials receive no activation tool. Sources: [scoped client construction](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgent.ts#L4180-L4205) and [scoped-session activation exclusion](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgentSession.ts#L2162-L2170).

## Activation flow

`CopilotComputerUseActivation.createTools()` is a product-specific bridge between runtime plugin status and a model-visible eager tool. It contributes `enable_computer_use` only to durable sessions, only while the built-in plugin is available and disabled, and only when policy and session mode permit an interactive choice. Sources: [`CopilotSessionLauncher` contribution point](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotSessionLauncher.ts#L316-L320), [tool assembly](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotSessionLauncher.ts#L1413-L1418), and [`CopilotComputerUseActivation`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotComputerUseActivationTool.ts#L125-L193).

The tool's `skipPermission: true` does not mean activation is silent: the tool owns a specific Turn On/Not Now card and treats acceptance as the permission step. It verifies the runtime result after `session.rpc.plugins.enable` instead of returning a success-shaped fallback. A decline suppresses repeat prompting within the relevant session state, and concurrent calls are serialized.

Activation persists for Copilot on the computer, including the Copilot CLI, rather than only for the current VS Code turn or process. Browser-use must deliberately choose whether activation is machine-wide, OS-user-wide, VS Code-profile-wide, browser-profile-wide, client-wide, session-wide, or per turn. Source: [activation card contract and persistence wording](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/common/meta/vscode/computerUse.ts#L193-L222).

The launcher seam itself is currently named `createComputerUseActivationTools`, so it is not yet a stable multi-plugin contribution API. Browser-use should generalize it only if agent-triggered activation is desired; a browser connection/onboarding surface may provide a clearer consent boundary.

## Permissions

There are three distinct permission decisions:

1. **Product enablement:** the user setting allows VS Code to resolve and inject the plugin.
2. **Plugin activation:** the explicit Turn On/Not Now card enables the trusted plugin in the Copilot runtime.
3. **Operation and operating-system access:** MCP approval governs tool calls, while macOS Accessibility and Screen Recording are handled through the plugin/helper's structured elicitation.

The specialized macOS flow is intentionally provenance-bound. `CopilotAgentSession` requires the exact built-in source before delegating to `CopilotComputerUsePermissionSetup`, whose wire contract uses namespaced root keys, schema versions, bounded strings/lists, nonces, and round identifiers. Sources: [provenance check and routing](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgentSession.ts#L8380-L8498), [shared root keys](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/common/computerUse.ts#L6-L16), and [validated wire readers](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/common/meta/vscode/computerUse.ts#L30-L157).

The setup service recognizes exactly two macOS privacy permissions and Computer Use's exact `request_*`, `settings_*`, and `try_again` actions. The renderer's helper-app drag UI is likewise specific to macOS Privacy & Security System Settings. Sources: [`CopilotComputerUsePermissionSetup` protocol constants/readers](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotComputerUsePermissionSetup.ts#L20-L155) and [`ComputerUseHelperAppContribution`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/workbench/contrib/chat/electron-browser/agentSessions/computerUseHelperApp.contribution.ts#L16-L36).

For MCP calls, host permission handling can approve a request when the trusted runtime recommends approval, based on the documented Computer Use assumption that the plugin separately gates each application. Managed approval and the assisted-approval judge still take precedence. Browser-use must not inherit that shortcut unless it has an equivalent, auditable browser/profile/site consent boundary; browser and page access are materially different from the current per-application assumption. Source: [MCP approval precedence in `CopilotAgentSession`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgentSession.ts#L7412-L7484).

## Concrete code seams

| Seam | Owning source and symbol | Status for browser-use |
|---|---|---|
| Local-only configuration mirroring | [`AgentHostConfigurationSyncScope.Local` mapping](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/common/agentHostConfigurationSync.ts#L109-L132) | Stable and reusable when the controlled resource is genuinely local. |
| Startup-only configuration fingerprint | [`CopilotAgentStartupConfig`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgentStartupConfig.ts#L9-L58) | Reuse the comparison/restart concept, but fingerprint an ordered plugin descriptor set rather than another one-off boolean/path. |
| Idle-safe runtime restart | [`CopilotAgent._requestClientRestart`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgent.ts#L2023-L2079) | Stable and reusable. |
| Trusted SDK injection | [`CopilotClientOptions.builtinPluginDirectories` construction](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgent.ts#L4248-L4267) and [SDK contract](https://github.com/github/docs/blob/3e6eca1bee3d0abed640682488ffe8600ebd32b0/content/copilot/how-tos/copilot-sdk/features/plugin-directories.md#L191-L206) | Stable SDK seam, but current VS Code wiring is singleton and must become aggregate. |
| MCP provenance and controlling setting | [`McpCustomizationController` metadata creation](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/shared/mcpCustomizationController.ts#L130-L155) and [plugin mapping](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/shared/mcpCustomizationController.ts#L563-L580) | Reuse provenance as-is; replace name-specific conditionals with descriptor metadata. |
| Defensive host/renderer wire contract | [`computerUse.ts` root keys](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/common/computerUse.ts#L6-L16) and [`meta/vscode/computerUse.ts` readers](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/common/meta/vscode/computerUse.ts#L30-L157) | Reuse the namespacing, versioning, bounds, and validation pattern if browser-use needs custom setup UI. |
| Staged pinned installation | [`CopilotComputerUsePlugin`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotComputerUsePlugin.ts#L120-L251) | Reuse as a pattern; extract a generic resolver rather than depending on the Computer Use-named class. |
| Activation contribution | [`CopilotComputerUseActivation.createTools`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotComputerUseActivationTool.ts#L125-L193) | Product-specific today; generalize only if browser-use needs model-visible activation. |
| Specialized permission setup | [`CopilotComputerUsePermissionSetup`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotComputerUsePermissionSetup.ts#L20-L155) | Not reusable as behavior; only its defensive protocol techniques are reusable. |

## Reusable versus Computer Use-specific pieces

### Reuse

- Configuration metadata that routes a gate to the correct Agent Host target.
- Independent locality checks in the owning process.
- Startup fingerprinting and deferred restart after active work drains.
- Trusted built-in plugin directory injection into the Copilot SDK.
- MCP provenance, source-plugin identity, and controlling-setting metadata.
- Exact release and per-target digest pinning.
- Cancellation-aware download, target-only extraction, staging, sentinel creation, atomic rename, and concurrent-winner checks.
- Namespaced, schema-versioned, bounded, validating wire contracts.
- Explicit failure results after activation RPCs rather than optimistic success.

### Reconsider or keep Computer Use-owned

- `CopilotComputerUsePlugin` hardcodes the plugin name, cache root, URL vocabulary, target hashes, log messages, and helper application; it is not a generic service. Source: [Computer Use constants and helper assumptions](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotComputerUsePlugin.ts#L41-L68).
- The client option currently receives a singleton plugin array, which cannot compose safely with a second integration. Source: [client option construction](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgent.ts#L4248-L4267).
- Activation persists across Copilot clients on the computer, including the CLI; browser-use needs a deliberate scope tied to its browser/profile model. Source: [activation persistence wording](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/common/meta/vscode/computerUse.ts#L193-L222).
- The macOS Accessibility and Screen Recording state machine, helper-app path, drag-to-System-Settings UI, and action vocabulary are Computer Use-only.
- Runtime-recommended MCP approval is justified by a Computer Use per-application gate and must not be generalized to browser pages or domains without equivalent enforcement. Source: [approval decision path](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgentSession.ts#L7412-L7484).
- Windows receives the plugin without the custom macOS setup service; Linux receives neither because the release has no pinned Linux target. Browser-use should define its target matrix independently.

## Failure handling

- Unsupported platform/architecture and unavailable installs do not block ordinary Copilot startup. `_resolveComputerUsePlugin` logs the unavailability and returns no path. Source: [`CopilotAgent._resolveComputerUsePlugin`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgent.ts#L1582-L1624).
- HTTP non-2xx, cancellation, checksum mismatch, extraction failure, and publication failure are explicit errors in the resolver. Cancellation tears down the response stream rather than waiting indefinitely. Sources: [download and checksum path](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotComputerUsePlugin.ts#L157-L218) and [publication path](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotComputerUsePlugin.ts#L220-L251).
- The outer integration catches resolver failures, logs that the plugin is unavailable, and preserves normal chat. There is no user-facing download failure/progress surface and no timed retry or exponential backoff. A failed install is retried only when a later Copilot client start resolves the plugin again.
- The activation tool checks policy, attended mode, prior decline, RPC failure, and post-enable runtime state, and reports failure instead of claiming activation succeeded. Source: [`CopilotComputerUseActivation` invocation](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotComputerUseActivationTool.ts#L125-L193).
- The permission setup rejects malformed, stale, duplicate, or wrong-round responses and falls back to generic elicitation when provenance or protocol does not match. Sources: [`CopilotComputerUsePermissionSetup`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotComputerUsePermissionSetup.ts#L292-L342) and [session routing](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgentSession.ts#L8380-L8498).

## Restart and update behavior

If installation finishes while the client is still starting, the just-started client is stopped and started again before exposure. If installation finishes after startup, the integration requests a quiet restart. Source: [post-install startup/restart handling](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgent.ts#L4284-L4304).

Turning the setting off cancels an active download and changes `CopilotAgentStartupConfig`, which requests a restart. `_requestClientRestart` waits for all turns and subagents to become idle before stopping the shared client. The setting description's "after a restart" therefore means an automatic Copilot runtime restart at the next safe point in the current implementation, not necessarily a full VS Code/window restart. Sources: [configuration-change handling](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgent.ts#L1983-L2002), [startup fingerprint](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgentStartupConfig.ts#L9-L58), and [restart scheduling](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgent.ts#L2023-L2079).

Plugin updates are entirely source-pin-driven:

- VS Code must change `COMPUTER_USE_PLUGIN_RELEASE`.
- The new version produces a new cache directory and downloads on the next eligible resolution.
- There is no latest-release query, periodic update check, in-place update, old-version pruning, or migration hook.
- Once `.complete` exists, startup does not re-hash or repair that version.
- A failed background install has no retry timer; another client start is the retry trigger.

Browser-use should preserve deterministic code pinning but add a manifest/digest record for completed-cache revalidation, safe stale-version pruning, bounded archive size, and a visible retry path if availability matters to the user.

## Remote/local boundary

The setting, resolver, and renderer each enforce locality independently. The local utility-process Agent Host chooses the archive for its own platform and architecture and stores it beneath its local `userDataPath`; it does not install into a remote extension host or remote Agent Host. Source: [configuration-sync test proving local-only targeting](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/test/common/agentHostConfigurationSync.test.ts#L267-L290).

The custom setup UI has a still narrower boundary: native local macOS with the `copilotcli` provider. Windows can receive and run the plugin but does not use this macOS setup service. Linux is unsupported by the pinned release table. Unknown or non-built-in elicitation follows the generic flow rather than crossing into the privileged setup path.

Scoped-credential Copilot clients form another boundary: they do not receive the built-in directory, and their sessions receive no activation tool. Sources: [scoped client construction](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgent.ts#L4180-L4205) and [scoped-session guard](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/node/copilot/copilotAgentSession.ts#L2162-L2170).

For browser-use, "local" must be defined across four resources: the Agent Host process, browser process, browser profile/data directory, and visible consent UI. Reusing the local setting scope is correct only when all four are intentionally co-located or when cross-host ownership is explicitly modeled.

## Test coverage and gaps

### Covered

| Area | Evidence |
|---|---|
| Wire request/status/card validation and schema-version fallback | [`computerUse.test.ts`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/test/common/computerUse.test.ts#L16-L143) |
| Download, cache hit, stalled-body cancellation, retry after cancellation, cancelled extraction, concurrent publication, incomplete-directory replacement, checksum mismatch, unsupported platform, helper path, and icon parsing | [`copilotComputerUsePlugin.test.ts`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/test/node/copilotComputerUsePlugin.test.ts#L70-L278) |
| Setting/local-host gate, restart, macOS-only status, background download, install-during-start, and cancellation on disable | [`copilotAgent.test.ts`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/test/node/copilotAgent.test.ts#L14602-L14777) |
| Activation availability, card behavior, policy, decline persistence, concurrency, and runtime failure | [`copilotComputerUseActivationTool.test.ts`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/test/node/copilotComputerUseActivationTool.test.ts#L89-L249) |
| Permission parsing, stale/duplicate requests, serial actions, cross-chat readiness, accessibility changes, helper icon, disposal, and generic fallback | [`copilotComputerUsePermissionSetup.test.ts`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/test/node/copilotComputerUsePermissionSetup.test.ts#L72-L606) |
| Runtime MCP recommendation handling and approval precedence | [`copilotAgentSession.test.ts`, approval cases](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/test/node/copilotAgentSession.test.ts#L10912-L10959) |
| End-to-end session wiring for activation, provenance, setup routing, and ordinary fallback | [`copilotAgentSession.test.ts`, integration cases](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/platform/agentHost/test/node/copilotAgentSession.test.ts#L24435-L24700) |
| UI modal/card behavior and restored-history projection | [`computerUseSetupDialog.test.ts`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/workbench/contrib/chat/test/browser/agentSessions/computerUseSetupDialog.test.ts#L74-L344) and [`stateToProgressAdapter.test.ts`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/src/vs/workbench/contrib/chat/test/browser/agentSessions/stateToProgressAdapter.test.ts#L612-L652) |
| Product packaging exclusion | [`build/lib/test/copilot.test.ts`](https://github.com/microsoft/vscode/blob/d0adb05e7252fd61c7a12d57681101a50b2d3de5/build/lib/test/copilot.test.ts#L538-L541) |

### Gaps

- No test fetches or attests the real official v0.1.95 assets and digests; installer tests use synthetic archives.
- No real Copilot CLI subprocess plus released Computer Use binary test.
- No macOS privacy-permission end-to-end test, Windows consent end-to-end test, or remote-window end-to-end test.
- No completed-cache corruption/tampering test because `.complete` bypasses revalidation.
- No direct coverage for HTTP non-2xx behavior, the two-minute startup timeout, an oversized-body limit, a user-visible retry, or retry backoff.
- No version-bump migration, stale-cache cleanup, or old/new helper interaction test.
- No test proves that two independent trusted built-in plugin directories are aggregated and preserved across startup and restart.
- The public `github/computer-use-plugin` v0.1.95 repository/release does not expose the MCP/helper implementation source, so helper replacement, app-consent, and signature semantics are observable only through VS Code's integration contract/comments and opaque release binaries.

## Risks and recommendations

1. **Define browser-use locality and consent first.** Specify which process owns the browser, which profile/data directory is used, which domains or tabs are in scope, and which actions require fresh human consent. Do not copy the Computer Use local-only gate or runtime-recommended MCP approval until those boundaries exist.
2. **Add a built-in-plugin descriptor registry.** Each descriptor should carry plugin ID, controlling setting, target matrix, release pin/digests, cache namespace, activation contribution, provenance, and optional setup contribution. Resolve enabled descriptors concurrently and build one deterministic `builtinPluginDirectories` array.
3. **Extract a generic pinned resolver.** Preserve target-specific hashes, cancellation, staging, atomic publication, and nonblocking startup; parameterize plugin ID, version, supported targets, URL, expected digest, cache key, extraction root, and optional post-extraction validation.
4. **Harden cache/update behavior.** Persist verified digest/manifest metadata, revalidate or repair completed caches, cap archive size, safely prune stale versions, and add bounded retry/backoff with a user-visible retry state.
5. **Keep Computer Use onboarding separate.** The macOS helper, Accessibility/Screen Recording state machine, drag-to-System-Settings UI, and persistent `/computer on|off` semantics should remain Computer Use-owned.
6. **Generalize activation only if browser-use needs it.** Replace `createComputerUseActivationTools` with generic built-in-plugin activation contributions only when model-visible activation is intentional; otherwise use explicit browser connection/onboarding UI.
7. **Preserve provenance checks.** Browser-use-specific elicitation or privileged UI must require both the expected built-in plugin provenance and a validated, versioned payload.
8. **Add release and multi-plugin tests.** Verify the pinned official release manifest/digests deterministically, test aggregation order and coexistence through restart, test completed-cache corruption, and add at least one supported-OS real-runtime smoke path.
9. **Align wording with runtime behavior.** Describe setting changes as causing a deferred Copilot runtime restart rather than implying that a full VS Code restart is always required.

## One-line gist for the map

Computer Use is a local-only, opt-in, hash-pinned trusted Copilot plugin with atomic background installation and quiet runtime restart; browser-use should reuse those host/SDK/provenance seams but replace singleton installer wiring and define its own locality, activation, update, and browser-consent model.
