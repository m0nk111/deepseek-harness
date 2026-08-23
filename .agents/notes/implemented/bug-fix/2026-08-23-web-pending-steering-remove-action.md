# Agent Note: Pending steering bubbles gain a remove action

Status: implemented

English | [中文](2026-08-23-web-pending-steering-remove-action.zh.md)

## Problem

While a turn runs, a steering message entered through the composer (or a queued
row's Steer action) renders at the conversation tail as a Host-authoritative
pending steering bubble. A queued row in the QueueDock offered edit, delete,
and strict-steer actions, but the pending steering bubble rendered only Copy.
There was no way to withdraw a steer once it was in the next-step window but
before the running turn admitted it — the user had to let the steer land and
then deal with it as durable history.

The Host surface already supported the operation: `conversation.updateQueue`
with `{ kind: 'remove' }` addresses a `next-step` occurrence exactly as it does
a queued one, retiring the occurrence from the pending set. The gap was purely
in the Web presentation.

## Decision

`PendingSteeringBubble` now renders a remove (trash) action beside Copy. Clicking
it calls the session-scoped `updateQueue` verb with `{ kind: 'remove' }`, which
the Host wires from the chat view's injected face. A rejection (the steer was
already admitted and its row has left the pending set) is silently swallowed: the
authoritative `session/queue` snapshot reconciles the bubble, so there is nothing
left to surface.

The action lives only on the pending, pre-admission bubble. A durable user or
steering bubble (rendered from `user/message`) keeps copy and clock and no trash
action, matching the existing rule that sent messages offer no removal.

A new locale key (`queue.removeSteering`) names the action distinctly from the
queued-row removal copy ("Remove queued message"), because the two surfaces
address different objects.

## Alternatives considered

- **Reuse the queued-row copy.** The queued-row label describes a queue message,
  while the pending bubble is a steer; the distinct object warrants distinct copy.
- **No-op on rejection instead of swallowing.** The authoritative snapshot already
  removes the row on admission, and surfacing an error for a benign convergence
  would add noise to a state that resolves itself.
- **Extend the durable bubble with removal.** Sent messages are durable model
  history; removing them would require host-side deletion semantics that do not
  exist. The affordance stays with the volatile pending occurrence it addresses.

## Consequences

A mid-turn steer can be withdrawn before it reaches the model, matching the
control a user already had over queued messages. The Web view gains one injected
verb (`updateQueue`) and passes it through to the pending bubble; no host or
session-log change is required.

## Testing

The chat-view component spec adds a pending steer and asserts clicking the
remove action calls `updateQueue` with the occurrence id and `{ kind: 'remove' }`,
while a durable user bubble never renders the action. The `apply-inject` spec
asserts the injected chat-view `updateQueue` reaches the session face. The two
`steering` web e2e `mid-steer` goldens regenerate to include the new button; the
`settled` goldens are unchanged because a durable bubble carries no remove action.
