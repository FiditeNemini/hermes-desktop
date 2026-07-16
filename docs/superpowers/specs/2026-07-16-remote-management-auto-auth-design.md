# Remote Management Auto Authentication Design

## Goal

Ensure the shared direct-Remote management client resolves the default `auto` authentication mode before choosing token or OAuth transport.

## Problem

`ConnectionConfig.remoteAuthMode` accepts `auto`, `token`, and `oauth`. The management request boundary currently treats every value except `oauth` as token mode, so an unresolved `auto` connection can send a session-token request to an OAuth-only dashboard and surface a generic 401.

The boundary's tests construct only explicit token and OAuth configurations, leaving the default mode uncovered.

## Design

`remoteDashboardRequestJson()` remains the single main-process transport boundary. It will resolve `auto` by calling the existing public `probeRemoteAuthMode()` status probe, then route the request using the returned explicit mode.

Explicit `token` and `oauth` configurations will skip the probe. Auto resolution will affect only the current request; it will not write connection configuration or expose authentication material to the renderer.

Resolved OAuth requests will continue through `requestRemoteOAuthJson()` and the persistent Electron cookie partition. Resolved token requests will continue through `remoteRequestJson()` and `X-Hermes-Session-Token` transport.

## Failure Behavior

Status-probe failures will pass through the existing management error normalization. The client will not guess a transport, retry with another credential type, or fall back to local state.

OAuth login-required errors will retain their existing reauthentication signal. Explicit-mode behavior and non-Remote rejection remain unchanged.

## Testing

Add focused regression coverage for both possible `auto` results:

- `auto` resolving to OAuth probes once, uses cookie transport, and never sends a token request.
- `auto` resolving to token probes once, uses token transport, and never sends a cookie-authenticated request.
- Existing explicit token and OAuth tests verify no regression in transport selection and profile scoping.

Run the focused request-client tests first, then the complete Vitest suite, node/web typechecks, and `lat check`.

## Documentation

Update the Remote Dashboard OAuth knowledge section so management authentication routing states that unresolved `auto` mode is probed at the request boundary and fails closed when detection fails.

## Delivery

Commit the regression test, implementation, and knowledge update on `pr/02-remote-management-client`. Push normally to `origin/pr/02-remote-management-client`; never force-push.
