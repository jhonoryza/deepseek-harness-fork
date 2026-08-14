# Agent Note: Trusted-LAN clients open the configuration plane via the host.describe privileged verdict

Status: implemented

English | [中文](2026-08-14-privileged-reachability-in-host-describe.zh.md)

## Problem

The Web GUI's configuration plane (Settings — Plugins section cards, the local-document action, the welcome acknowledgement, and the produced-files folder action) was gated client-side on `connection.isLoopback`. That is a page-authority heuristic, invisible to the server's trust fence: on a trusted-home-LAN deployment the server already admits privileged methods from the declared `trustedHosts` when `allowPrivilegedFromTrustedHosts` is set (the configuration-plane boundary [who may reach it](2026-07-30-config-plane-boundaries.md)), yet the browser, reached as `http://192.168.x.x`, still classified itself remote and rendered the Plugins configuration panel empty (a `persistence: 'memory'` scope never reads the Host settings document). The server fence — `isTrustedApiRequest` per request Host — was the only correct source of truth, but client plugins receive no cordis config, so the client could not know its own verdict.

## Decision

The connection layer annotates the readiness handshake's `host.describe` response with the request's own privileged verdict. In the node-half `/api` fallback, `privilegedReachable = isTrustedApiRequest(request, allowPrivilegedFromTrustedHosts ? trustedHosts : [])` — the same predicate that gates `PRIVILEGED_METHODS` — and a successful `host.describe` envelope has `result.value.privilegedReachable` set to it (non-success or non-envelope responses pass through untouched). The field is optional on the schema and on the contract type, so in-process handler paths that never annotate it keep working and clients fall back to `isLoopback`.

Client consumers read the verdict reactively through the existing `connection.hostDescription` channel (the handshake runs after streams open, after UI plugins have already applied — `bind()`/`apply()` construct with the pre-handshake fallback, then a `hostDescription.subscribe` promotes on the first annotated description):

- `SettingsScopeController` gains `upgradeToHost()`; `SettingsScopeBinder.bind` constructs with `hostDescription.getSnapshot()?.privilegedReachable ?? isLoopback` and subscribes to promote a memory scope to Host persistence, which starts the real `settings.describe` read.
- `WelcomeNoticeStore` gains `upgradeToHost()`; the ui-settings-models apply wires the same reactive construction and promotion.
- ui-settings-general drops the loopback gate and always registers the open-document action: a privileged 403 degrades `SettingsDocumentStore.load()` to `unavailable` and the action renders null, so an unprivileged remote browser simply sees no action.
- ui-deliverables' `ProducedFiles` computes `canOpenPath = (privilegedReachable ?? isLoopback) && hostCanOpenPath` through `useHostDescription`.

Default safety is unchanged: an unprivileged remote browser keeps memory persistence and the unavailable/unrendered states, never a permanent spinner — `upgradeToHost` is the only transition, and it only fires on a server-annotated `privilegedReachable: true`.

## Alternatives considered

- **A new ConnectionHandle API exposing the verdict** (a method or a dedicated source): rejected — `hostDescription` already carries the handshake value and its subscribe channel; the optional schema field rides the existing type with no new surface.
- **Computing the verdict in the apiproxy handler**: rejected — `toFetchHandler` and the `ApiProxy` implementation are request-agnostic (they see no Host header), so only the connection plugin's fallback closure has both the request and the config.
- **Making the schema field required**: rejected — exact-object tests and in-process handler paths (fixture/`InProcessApiClient`) would break for no wire benefit; absent stays meaningful as "not annotated".

## Consequences

- Trusted-LAN browsers (server `allowPrivilegedFromTrustedHosts: true` + trusted authority) now render the full Plugins configuration, local-document, welcome, and produced-files surfaces, because the server's own fence verdict reaches the client.
- The client keeps one reactive source of truth: page authority pre-handshake, server verdict after; reconnects republish the description per generation.
- Untrusted-remote behavior is byte-for-byte the previous posture (memory/unavailable), including the 403-on-describe path, which now degrades to a hidden action rather than an absent registration.
- The host wire gains one optional boolean on `host.describe`; no protocol version bump (client and host ship together).
