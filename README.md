# SignalingFakeTimeProvider Design Documentation

## Overview

The `SignalingFakeTimeProvider` is a specialized test double that extends `FakeTimeProvider` (from `Microsoft.Extensions.Time.Testing`) to solve a critical synchronization problem in asynchronous timer-based test scenarios. It intercepts timer creation via `CreateTimer` and signals a `TaskCompletionSource` each time a new timer is registered, enabling deterministic test coordination with the SQS message visibility renewal loop.

## Table of Contents

1. [Overview](README.md) - This file
2. [Component Design](component-design.md) - Detailed design of all components, classes, and interfaces
3. [Class Diagrams](class-diagrams.md) - PlantUML class diagrams showing inheritance and relationships
4. [Sequence Diagrams](sequence-diagrams.md) - PlantUML sequence diagrams showing runtime interactions

## Problem Statement

The `Renewal.RenewMessageVisibility` method runs an asynchronous loop that:

1. Calculates the next renewal time
2. Calls `Task.Delay(renewAfter, timeProvider, cancellationToken)` which internally calls `TimeProvider.CreateTimer()`
3. Awaits the delay, then renews the message visibility via the SQS API
4. Repeats until cancelled

When testing with `FakeTimeProvider`, advancing time triggers timer callbacks. However, there is a race condition: after advancing time and triggering a renewal, the code must register the *next* timer before the test can advance time again. Without synchronization, the test may advance time before the next timer is registered, causing missed renewals and flaky tests.

## Solution

`SignalingFakeTimeProvider` overrides `CreateTimer` to signal a `TaskCompletionSource` each time a new timer is created. Tests call `WaitForNextTimer()` before advancing time, then `await` the returned task to guarantee the renewal loop has progressed and registered its next timer.

## Key Design Decisions

- **`volatile` field**: The `_nextTimerRegistered` field is marked `volatile` to ensure visibility across threads without requiring locks.
- **`RunContinuationsAsynchronously`**: Both the field initializer and `WaitForNextTimer()` use `TaskCreationOptions.RunContinuationsAsynchronously` to prevent deadlocks from synchronous continuations running on the signaling thread.
- **`TrySetResult()`**: Uses `TrySetResult()` rather than `SetResult()` in `CreateTimer` to be safe against multiple timer creations without a matching `WaitForNextTimer()` call.
- **Nested class scope**: Defined as a nested class within `RenewalTests` since it is specific to renewal test scenarios and not needed elsewhere.
