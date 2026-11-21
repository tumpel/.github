# RFC: Foreman, a session that turns its crew's events into its own turns

| Field   | Value                                                                                                                                                                                                                                                                                                                     |
| ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Status  | Proposed                                                                                                                                                                                                                                                                                                                  |
| Scope   | Design only; the tenant Durable Object, the public API, MCP, the CLI, the connector token endpoint and the event catalogue                                                                                                                                                                                                |
| Related | [RFC: Events](RFC-TUM-Events.md), [RFC: Connectors](RFC-TUM-Connectors.md), [RFC: Tenant API](RFC-TUM-TenantApi.md), [ADR 007: turn goal](007-turn-goal-on-harness-goal-feature.md), [Claude Code goal docs](https://code.claude.com/docs/en/goal), [EDD-006](006-end-to-end-proof.md) |

Storage follows [ADR-010](010-bulk-data-out-of-the-database.md). The database holds metadata only, and bulk data lives in R2.

## Summary

A foreman is an ordinary harness session with a plan, a goal and a list of events to react to. Each time one of those events happens, Tumpel sends the foreman a new turn. The turn holds the plan, the event and the goal. The foreman picks the next step, such as a session on the next issue. A goal keeps it working, and it calls `Subscriptions_complete` once the goal holds. A foreman that needs a decision pauses with a short message and waits for a reply.

The foreman reacts only to its crew: the sessions that carry the tags its subscription names. Tumpel records which session started each session, so every session the foreman starts gets those tags on its own. Every subscription has an expiry time, which caps how long it runs. Tumpel pauses it after 10 crew failures in a row.

## Problem statement

1. The owner's brief of 2026-11-19 asks for a session that works through a long goal, such as an issue list, while nobody watches.
2. An event only reports state. RFC: Events R6 rules that an event never starts work, and its out-of-scope list names "an event that starts a turn".
3. A session holds no tags. The `sessions` table has no such column, and only hosts carry labels.
4. A session records no origin. A call from inside a box reaches the Tumpel MCP server through the connector forwarder with the access token of the connector's OAuth grant (RFC: Connectors D2, D9). The Worker's `Caller` holds a tenant, a principal and a role, so two sessions on one grant look the same.
5. Tumpel learns no goal verdict. Claude Code records a met goal only in its transcript ([goal docs](https://code.claude.com/docs/en/goal)), and ADR 007 forbids the control plane to read the transcript to judge a goal.
6. A session has no way to ask a person a question. [Public surface](surface.md) lists no such operation.

## Requirements

1. R1: A foreman is a plain harness session with a subscription, and nothing else. The subscription holds the plan. Each matching event becomes a new turn with the event attached. Source: owner's brief, 2026-11-19, and review.
2. R2: A foreman hears only its own crew. Source: owner's brief.
3. R3: Sessions carry flexible tags that serve other uses too. Tags are the preferred way to scope a crew. Source: owner's brief.
4. R4: A session's origin records the session that started it, so the session inherits its crew tags. Source: owner's brief and rulings.
5. R5: An expiry time and a pause after 10 consecutive crew failures stop a foreman that reacts to its crew without end. The expiry, measured in days, is the runtime limit. Source: owner's brief and review.
6. R6: Tenets 1, 2 and 8 hold: the guest stays unchanged, no transcript decides a delivery, and no Durable Object waits.
7. R7: Every new operation follows [API](api.md) rule 1 and 5.
8. R8: A foreman always runs with a goal and ends its subscription with an explicit completion signal. Source: owner's review and rulings.
9. R9: A foreman can pause itself with a short note that reaches the person as a notification. A reply turn and a resume restart it. Source: owner's review.
10. R10: The Tumpel MCP server's instructions and tool descriptions advertise subscriptions, so an agent handed a long job learns that it can set up a foreman. Source: owner's review.

## The foreman and its subscription

Decision D1: "foreman" is this RFC's name for a session with one or more Active subscriptions. No type, table, field or enum value names a foreman. The session runs any persona. `Subscriptions_create` gives it the Tumpel MCP (D8), so it can start sessions and send turns through the existing MCP tools.

Decision D2: a subscription holds these fields. Every field is mandatory ([Data modeling](data-modeling.md) rule 3). Subscriptions live in their own tenant table.

| Field                    | Meaning                                                                         |
| ------------------------ | ------------------------------------------------------------------------------- |
| `subscriptionId`         | Its key                                                                         |
| `sessionId`              | The session the subscription sends its turns to                                 |
| `prompt`                 | The plan, sent at the top of every delivery turn                                |
| `goal`                   | The condition that ends the work, sent with every delivery turn (D16)           |
| `events`                 | The event types it reacts to, one or more, each a session event                 |
| `crewTags`               | The tags that make a session part of the crew. It must carry every one          |
| `expiresAt`              | The runtime limit: an ISO 8601 time with offset, after which it ends            |
| `cursorEventId`          | The last event it has delivered                                                 |
| `crewFailuresInARow`     | Crew turns and sessions that failed since the last crew turn completed (D13)    |
| `deliveryFailuresInARow` | Delivery turns that failed since the last one completed (D12)                   |
| `state`                  | One model per state, each with its reason ([TypeSpec](typespec.md) rule 6), listed below |

The three states and their reasons:

1. `Active`.
2. `Paused`, with reason `ByRequest` and its `message`, `DeliveryFailed` and the last failed `turnId`, or `CrewFailing`.
3. `Ended`, with reason `SessionArchived`, `SessionFailed`, `Expired` or `GoalMet`.

A session event is one whose payload holds a `sessionId`.

## Lifecycle

Decision D3: these operations manage subscriptions. Each has a `tumpel_subscription_*` MCP tool and a `tumpel subscription` CLI command.

| Operation                | Result                                                                  |
| ------------------------ | ----------------------------------------------------------------------- |
| `Subscriptions_create`   | Creates an Active subscription. Its cursor starts at the end of the log |
| `Subscriptions_list`     | Lists them, filtered by `sessionId` or state                            |
| `Subscriptions_get`      | Reads one, with its failure count and its reason                        |
| `Subscriptions_pause`    | Moves it to Paused, reason `ByRequest`, with a `message` (D20)          |
| `Subscriptions_resume`   | Moves it to Active and delivers from its cursor                         |
| `Subscriptions_delete`   | Deletes it and cancels its waiting delivery turn                        |
| `Subscriptions_complete` | Ends it with reason `GoalMet` (D18)                                     |

Decision D4: only a person creates or resumes a subscription. A call through a person's own grant counts, also when the person asks an assistant to make it. A call that carries a turn token (D8) gets 403 on both, so no foreman starts or revives a foreman. An agent inside a box that wants a subscription tells the person, who sets it up. The foreman can pause, delete or complete its own subscription.

Decision D5: a subscription ends when its session is archived or fails, when `expiresAt` passes, and when the foreman completes it (D18). A new `SubscriptionExpiry` deadline on the session's existing `SessionTimer` ends it on time.

## Tags

Decision D6: tags group sessions in any combination, apart from host labels. A tag is a lower-case string matching `^[a-z0-9][a-z0-9:._-]{0,62}$`. Each session holds up to 16. The tenant table `session_tags (session_id, tag)` stores them, with that pair as its primary key, so a crew match filters in the query ([Data modeling](data-modeling.md) rule 10).

The tag operations take the shape of the host label operations (`Hosts_setLabels`, `Hosts_patchLabels`):

1. `Sessions_start` and `Sessions_startFromIssue` take an optional `tags` input, an additive change ([API](api.md) rule 3).
2. `Sessions_setTags` replaces a session's tags. `Sessions_patchTags` takes `add` and `remove`.
3. `Sessions_list` takes a repeatable `tag` filter.
4. Every session read returns `tags`.

The crew is whatever carries the tags at the moment of the event. Removing a tag takes that session out of the crew for every later event.

## Origin

Decision D7: a session's origin is either a person or a turn of another session. `Session.origin` is one of two models: `PersonOrigin` with the `principalId`, or `TurnOrigin` with `sessionId` and `turnId`. Its one use is tag inheritance (D9). The tenant table `session_origins (session_id, origin_session_id, origin_turn_id)` holds the second case, so no column is nullable.

Decision D8: the connector token endpoint already knows the turn behind each request. Tumpel adds a built-in MCP connector with the fixed id `tumpel`, the stage's own MCP endpoint and no stored credential. A subscribed session gets it, and any other session gets it when its persona sets `tumpelMcp`. For it, the endpoint returns a turn token. The Worker signs it. It names the tenant, the grant's principal, the session and the turn, and it expires when the turn ends. Every Tumpel call that carries it records its origin. The forwarder sends whatever token it gets, so the guest stays unchanged.

Decision D9: a session started with a turn token inherits the `crewTags` of every Active subscription of the starting session.

## Delivery

Decision D10: matching runs inside `appendEvent`, in the same transaction that writes the event. A subscription matches an event when all four hold: the subscription is Active, it lists the event type, the event's session carries every crew tag, and that session is neither the foreman nor a session with an Active subscription of its own.

On a match, the tenant Durable Object makes sure the subscription has one delivery turn waiting on its session. It inserts that turn into `turn_session_waits` when none waits. After commit, it starts the turn through the path that starts a queued turn when the foreman is idle. A busy foreman starts it when its running turn ends, in send order, as `takeNextWaiting` does today. A stopped foreman restores its box first, as it does for any turn sent to a stopped session.

Decision D11: the control plane writes the delivery turn's prompt when the turn starts. The prompt is the plan, then every matching event after the cursor in `eventId` order, one JSON line each, up to 100 (the event page size). A long plan and goal shrink the batch, because the fold caps the message at 4,000 characters (D17). Tumpel records the last event included with the turn. When more events wait, a second delivery turn waits behind the first.

So events that arrive while the foreman works join one waiting turn. Events reach the foreman in log order.

## Failures

Decision D12: the cursor moves to a delivery turn's last event when that turn completes. A failed delivery turn leaves the cursor where it was and adds one to `deliveryFailuresInARow`. Tumpel then queues the next delivery turn at once, and it holds the missed events plus any new ones. A completed delivery turn resets the count. At 3, Tumpel pauses the subscription with reason `DeliveryFailed`. A person reads it with `Turns_getFailure` and resumes.

Matching and the event commit in one transaction, so a deploy or an eviction loses no delivery.

The event log keeps 7 days (RFC: Events D2). A subscription paused for longer has a cursor older than the log, and `Subscriptions_resume` fails 410 `SubscriptionCursorExpired`. Its fix tells the person to delete it and create a new one.

## Loop bounds

Decision D13: four rules bound a foreman.

1. A foreman never hears its own session, so its own `TurnCompleted` starts no delivery.
2. Events of any session that holds an Active subscription match no subscription, so two such sessions in each other's crew never trigger each other.
3. Each `TurnFailed` and each `SessionFailed` of a crew session adds one to `crewFailuresInARow`, and each `TurnCompleted` of a crew session sets it back to 0. The subscription counts both events even when its `events` list leaves them out. At 10, Tumpel pauses it with reason `CrewFailing`. Resume sets the count to 0.
4. `expiresAt` ends it (D5).

Decision D14: a subscription has no dollar cap. The expiry bounds what it spends.

## Events

Decision D15: this RFC changes [RFC: Events](RFC-TUM-Events.md). R6 becomes: an event starts a turn only through a subscription a person created. That meets tenet 3 (D4). The catalogue adds `SubscriptionPaused` and `SubscriptionEnded`, each with `subscriptionId`, `sessionId` and `reason`. Both belong to the foreman's own session, so rule 1 of D13 keeps them from the foreman.

## Goal

Decision D16: every subscription holds a mandatory `goal`, and each delivery turn carries it through the turn goal path of [ADR 007](007-turn-goal-on-harness-goal-feature.md). Tumpel stores the goal in `turn_goals`. At the offer, the control plane folds the prompt and the goal into one `/goal` message, in the form the harness catalog names. The harness's evaluator then judges the goal after each harness turn (goal docs).

Delivery turns stay short: their fold caps the goal loop at 5 rounds, in place of `GOAL_TURN_CAP` (20). Events that arrive meanwhile wait for the next delivery turn. D23 tells the foreman to delegate heavy work.

Decision D17: `Subscriptions_create` refuses a session whose harness takes no goal, with the refusal that `Turns_send` gives. Only Claude Code takes a goal today. It also refuses a plan and a goal that leave no room for one event inside the 4,000-character fold limit.

Decision D18: the foreman calls `Subscriptions_complete` with its turn token when its goal holds. The subscription moves to Ended with reason `GoalMet`. Tumpel cancels its waiting delivery turn and appends `SubscriptionEnded`, which reaches the person through the paths of D21. Crew sessions go through the existing idle stop and archive like any session.

## Completion

Decision D19: Tumpel never reads a harness verdict. The goal feature keeps the foreman working inside each delivery turn, and only `Subscriptions_complete` ends the subscription with `GoalMet` (tenet 2, problem 5).

## Pause with a message

Decision D20: `Subscriptions_pause` takes a `message` of up to 500 characters. The foreman calls it with its turn token on its own subscription when it needs a decision. Its `ByRequest` reason holds the message. Pausing cancels the subscription's waiting delivery turn, whose events stay after the cursor. New events wait there too.

Decision D21: `SubscriptionPaused` carries the reason, so a `ByRequest` pause carries its message to the person. The dashboard stream and `tumpel events follow` carry it now. Webhooks and Web Push take it to a phone once their slices land (RFC: Events D7, D8, slices E2 and E3).

Decision D22: the person replies with an ordinary `Turns_send` to the session, then calls `Subscriptions_resume`. That reply turn sits first in the session's queue, so the foreman reads it before the next delivery turn (D10).

## Guidance in the MCP server

Decision D23: the Tumpel MCP server tells every agent that drives it about subscriptions. The MCP server carries both texts:

1. `INSTRUCTIONS` is the server's built-in guide, returned in `instructions` by the initialize and discover results. It adds a step: for a long job, start a session and give it a goal and a subscription with `tumpel_subscription_create`. It tells a subscribed session to give heavy work to its crew and to call `tumpel_subscription_complete` at the end. An agent in a box tells its person instead (D4).
2. Each entry of `TOOLS` carries its tool description, through the `rest` helper or inline. `tumpel_session_start` and `tumpel_session_start_from_issue` point to `tumpel_subscription_create` for long jobs. Each `tumpel_subscription_*` description says what it does and who can call it (D4).

## Delivery plan

Decision D24: each slice is one PR. Protocol slices release first, and the CLI releases before the control-plane slice that needs its commands, because the surface check reads the pinned CLI (RFC: Events D10).

| Slice | Component       | Change                                                                                                            |
| ----- | --------------- | ----------------------------------------------------------------------------------------------------------------- |
| P1    | protocol        | Tags and origin models, the tag operations and filters, the persona's `tumpelMcp`                                 |
| C1    | CLI             | `tumpel session set-tags` and `patch-tags`, the `--tag` flags, the persona's `tumpelMcp`                          |
| T1    | control plane   | `session_tags`, `session_origins`, the built-in Tumpel MCP for opted-in personas, the turn token                  |
| P2    | protocol        | Subscription models with the goal, the seven operations, the pause message, the two events, the new problem codes |
| C2    | CLI             | `tumpel subscription` commands                                                                                    |
| T2    | control plane   | Subscriptions, matching, delivery turns with the 5-round fold, delivery retries, crew failures, expiry            |
| T3    | control plane   | The Tumpel MCP for subscribed sessions, tag inheritance, the pause message, completion                            |
| T4    | control plane   | The MCP instructions and tool descriptions (D23)                                                                  |

The proof in EDD-006 adds one step: a foreman with one tag and a goal starts one crew session and receives its `SessionIdle` as a turn. It then pauses with a message, gets a reply, is resumed, and calls `Subscriptions_complete`.

## FAQ

### Can a session with a subscription create another one?

Subscriptions come from people. A call with a turn token gets 403 on create and resume (D4).

### Does Tumpel judge whether the plan is done?

The foreman does, with `Subscriptions_complete` (D18, D19).

### Can the foreman stop and wait for a decision?

It calls `Subscriptions_pause` with a message. A reply through `Turns_send` and a resume restart deliveries (D20, D22).

## Appendix of terms

| Term           | Meaning                                                                             |
| -------------- | ----------------------------------------------------------------------------------- |
| Foreman        | This RFC's name for a session with an Active subscription; no type carries it       |
| Crew           | The sessions that carry every tag a subscription names                              |
| Subscription   | A plan, a goal, a list of event types, crew tags and an expiry, tied to one session |
| Delivery turn  | A foreman turn that a subscription queued                                           |
| Turn token     | A token the Worker signs for one turn, so Tumpel knows which session called         |
| Origin         | The person or the turn that started a session                                       |
| Cursor         | The last event a subscription delivered                                             |
| Goal evaluator | The separate model inside the harness that keeps the goal loop going                |
