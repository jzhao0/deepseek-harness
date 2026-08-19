# Agent Note: Subagents inherit the parent's live request route

Status: implemented

English | [中文](2026-08-19-subagent-live-route-inheritance.zh.md)

## Problem

In-process subagent creation inherited `provider`, `model`, and `maxTokens` from `parent.options`. Those options describe how the parent Agent was created, not necessarily the route used by its next request. A parent that switched provider or model during the session therefore delegated a new child onto its stale creation-time route.

The defect affected both local one-shot delegation and continuable creation. One-shot children were materialized with stale `AgentOptions`. Continuable creation also derived the durable descriptor's `agentProvider` and `agentModel` from the stale parent options, so the wrong route was persisted before the child was materialized.

The Session already owns the authoritative incremental request-route snapshot: `session.requestHeader()` returns the header in force for the next request. Child routing needs the same delegation-time view.

## Decision

In-process child route inheritance resolves the parent's latest `session.requestHeader()?.config` at the delegation boundary. `provider`, `model`, and `maxTokens` come from that live request config when present and fall back independently to `parent.options` before a corresponding live value exists. Explicit per-child `agentOptions` retain precedence over every inherited value.

`resolveChildAgentOptions()` owns this rule for both delegation paths.

Continuable creation resolves `AgentOptions` exactly once before its first await. The durable descriptor reads `provider` and `model` from that resolved snapshot, and later materialization reuses the same object instead of resolving the parent route again. Descriptor persistence and runtime child creation therefore describe one delegation-time route even if the parent changes after delegation begins.

Cold resume does not re-read the parent's current route. It continues to reconstruct the child from the route persisted in that child's descriptor, so later parent switches do not retroactively change an established durable child.

## Alternatives considered

**Keep inheriting from `parent.options`.** Rejected because creation options intentionally remain stable while request routing may change during the Agent's lifetime. Treating them as current routing state reproduces the stale-provider defect after every mid-session switch.

**Update `parent.options` whenever the parent route changes.** Rejected because it would turn creation metadata into mutable request state and broaden a subagent bug fix into Agent-wide option semantics. The Session already records the current request configuration explicitly.

**Resolve the route separately for the continuable descriptor and child materialization.** Rejected because an await separates those operations. Re-reading the parent could persist one route and materialize another if the parent changes in between.

**Copy only provider and model from the live header.** Rejected because `maxTokens` is part of the same inherited request route. Splitting its source from provider/model would leave child request configuration internally inconsistent.

## Consequences

A child delegated after a parent provider/model switch now follows the route in force for the parent's next request rather than the route used when the parent Agent was created. Before the first request header exists, behavior remains compatible through the per-field fallback to creation options. Explicit child overrides are unchanged.

Continuable descriptors now persist the same resolved provider/model snapshot used to create the child. Cold resume remains stable because it reconstructs that persisted child route rather than sampling the parent's later state.

Regression coverage pins both paths with a parent whose creation options remain `mock/mock` while its Session advances from an initial request header to `live/live-model` with a different `maxTokens`. The one-shot child must materialize with the live route; the continuable descriptor and child request header must both carry the same live snapshot.
