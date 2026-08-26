---
layout: post
title: "ESS Architecture Guide"
date: "2026-08-26 04:52:44 +0000"
tags: ["architecture", "events-states-services", "guide", "agents"]
---

# Events, States, Services Architecture Guide

Status: Proposed house architecture for agent-coded projects.

## Intent

Use a small shared vocabulary so implementation agents and reviewer agents can reason about architecture consistently across projects without forcing every project into the same shared framework.

## Core Loop

1. An adapter receives external input.
2. The adapter parses and validates transport shape.
3. A service performs business logic against explicit state or persistence boundaries.
4. State changes are owned by one obvious owner.
5. Typed events or typed results report what happened.
6. Adapters format the result for humans, tools, UI, CLI, HTTP, MCP, LLMs, or storage.

## Primitives

### State

State is mutable truth owned by a clearly identified boundary. It may be a live `*State` object, a persisted artifact, a database schema plus repositories, a `*View` or `*Snapshot` derived from another owner, a service-owned `*Flow`, or an operation-scoped `*Workspace`.

Rules: one writer-owner; durable state has persistence; derived views/indexes/caches have rebuild behavior; flows/workspaces have reset/dispose/replacement behavior; observers use read-only views, snapshots, service methods, or events.

### Service

Service is the behavior boundary. It owns validation, sequencing, orchestration, and business rules.

Default: services should be stateless when practical and depend on explicit state/persistence boundaries.

Allowed exception: services may own mutable state when they are the clear single owner and the state is classified as persisted authority, derived rebuildable state, or ephemeral coordination state.

If a service owns state, document who owns it, whether it is saved, how it rebuilds, and when it clears.

### Event

Event is a typed fact that happened. Use events for indirect cross-component notification. Avoid raw `Action`, string commands, or transport DTOs as internal contracts. Infrastructure may be an event bus, returned event records, appended stream entries, or project-specific typed signaling.

If string event kinds are required for config/API compatibility, centralize constants and validation.

### Adapter

Adapter is a boundary to the outside world or a specific runtime: UI, CLI, HTTP, MCP, stdio, LLM provider, persistence, notification transport.

Adapters parse input, call services, publish/receive typed events, and format output. They should not own durable business state or core business logic.

### Commands

Avoid unqualified `Command` in architecture prose. Say which kind:

- `IntentCommand`: named request to do something.
- `UndoableCommand`: reversible user mutation.
- `CliCommand`: CLI/debug command adapter.
- `ToolCommand`: MCP/LLM/tool adapter command.
- `SqlCommand`: database command local concept.

### Registry

Registry is an explicit registration map, not mutable business state. Registration should be visible and reviewable.

### Store / Repository

Store or repository is a persistence/cache adapter. Pick one convention per repo. Do not rename mature repository projects just to match the guide.

### Context

Context is a temporary invocation envelope. If it persists, use `Snapshot`, `Envelope`, or `Record`. If it coordinates a multi-step process, use `Flow`. If it mutates during one operation, use `Workspace`. If it owns authoritative mutable data, use `State`.

## Authority Rules

- One mutable owner.
- Explicit lifecycle.
- Typed internal contracts.
- Thin adapters.
- Shared docs before shared modules.

Extract shared code only when behavior repeats across projects, has independent tests, and removes more complexity than it adds.