# Agent Note: Project pwsh escalation schema from the calling session

Status: implemented

English | [中文](2026-08-29-session-scoped-pwsh-escalation-schema.zh.md)

## Problem

The native OpenAI Responses route produced a deterministic failure before a harmless PowerShell command ran. At a session already using `danger-full-access`, the model exhaustively filled optional arguments with their defaults and requested `sandbox_permissions: "danger-full-access"`. The execution guard correctly rejected that as not strictly wider, but the registry-global schema had advertised an impossible option to the calling session. Prompt prose alone had not stopped repeated default placeholders such as an empty `workdir`, `run_in_background: false`, or the redundant escalation pair.

The [subprocess sandbox decision](../feature/2026-07-06-sandbox.md) intentionally kept a global escalation vocabulary because registration-time defaults cannot describe per-session overrides. Re-registering tools per session remains the wrong ownership model, but native request assembly now carries the exact calling agent and therefore has enough information to present a narrower model surface without changing the registry.

## Decision

In a confining composition, `dsh-tool-pwsh` retains one registry-global definition with the complete `ESCALATION_TARGETS` vocabulary. During `system-prompt/assemble`, after downstream assembly completes, the plugin verifies that the visible `pwsh` definition is its own registration, resolves the calling session through `ctx.sandboxPolicy`, and copies only that definition with `WIDER_MODES[effectiveMode]`:

- `read-only` exposes `workspace-write` and `danger-full-access`.
- `workspace-write` exposes only `danger-full-access`.
- `danger-full-access` removes `sandbox_permissions`, `justification`, and escalation-only prose.

Agent-less diagnostic assembly retains the registry baseline because it has no session truth to project. Request-local copies prevent concurrent agents with different policies from mutating or observing one another's definitions. Scoped replacement or hiding of `pwsh` is also preserved because projection proceeds only when the visible definition is the exact registered tool.

Every optional field now begins with its omission semantics. The descriptions instruct the model to send only `command` and `description` for ordinary foreground calls, to send `run_in_background` only as `true`, and to send escalation fields only after an actual sandbox denial. These descriptions also reach PTC-mode SDK text rendered from a confining composition's registry baseline.

Execution remains the safety boundary. `approveEscalation()` independently checks strict widening against the call's resolved effective mode before approval or process start, so stale schemas and injected arguments still fail closed.

## Alternatives considered

**Add prompt prose only.** Rejected because the existing tool description already prohibited speculative escalation, yet the model still populated every optional property.

**Silently discard a non-widening escalation pair.** Rejected because executing under standing wider access after ignoring the model's requested semantics would hide a malformed call and weaken the audit trail.

**Re-register a separate tool schema for every session.** Rejected because registry identity is process-wide and stable. Request-local projection supplies the needed model surface without mutable registrations.

**Require a machine-matched prior denial before exposing escalation.** Deferred because command identity across quoting, workdir, environment prefixes, and pipelines is fragile. The current denial hint, request description, human approval, and execution guard remain the enforcement choreography.

## Verification

Focused unit coverage assembles read-only, workspace-write, and danger-full-access agents concurrently, pins the agent-less registry baseline, and injects an exact danger-full-access-to-danger-full-access call to prove it neither prompts nor executes. The package build and type-aware lint pass. The keyless Windows `pwsh-tool-turn` replay runs the real sandboxed PowerShell executor at danger-full-access and snapshots a request schema with no escalation pair.

## Consequences

The direct Native `pwsh` schema no longer offers a same-mode escalation that the harness must reject. In a confining composition, registry inspection, agent-less diagnostics, and PTC-mode SDK text intentionally retain the complete vocabulary; a same-mode PTC call and any other untrusted input still fail at the execution guard.

A session sandbox-mode switch can change the native request's tool header while `pwsh` remains visible, invalidating KV Cache reuse from the first changed schema token. This is accepted because mode switches are rare, while advertising an impossible optional value caused deterministic user-visible failures. The implementation deliberately applies only to `pwsh`; other tool families keep their existing schema decisions until equivalent evidence justifies a separate change.
