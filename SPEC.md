# Agent HQ: platform spec

Agent HQ is a small, self-hosted control room for a personal team of AI agents. One person (the
**owner**) works with a **chief of staff** agent and a few **area leads**. The agents do the work, and the
owner makes the decisions that only they can make, from one queue on their phone or desktop. Everything
lives in one app: the owner's standing judgment (the **brain**), the decision **queue**, the **work** the
agents are doing, and the owner's knowledge organized by **area**.

This document describes the platform: the data model, the API, the agent protocol and the deploy flow.
It contains no instance-specific names, domains or ids. Those live in an instance's own configuration and
`CLAUDE.md`.

## Principles

- **One place for the owner.** Agents never ask in chat. Anything that needs the owner is a queue item,
  readable on a phone in under a minute, with a recommendation and 2-4 options.
- **Discretion.** Agents post only what is blocked on the owner or needs their eyes before it ships
  (production, personal data, money, merges). Status goes on the agent's card, not in the queue.
- **Production only through approval.** Agents never deploy or merge directly. Shipping is an item the
  owner approves; a workflow merges and ships exactly the commit that was approved.
- **Nothing is deleted by agents.** History is kept: decisions, the brain's versions, task events,
  superseded screenshots.
- **Every miss gets a prevention.** A rule, a check or a code change, written down where it happened.

## Roles

| Role | What it does |
|---|---|
| Owner | Decides queue items, edits the brain, adds tasks, comments, captures thoughts into the inbox. |
| Chief of staff (`cos`) | The owner's main interface. Turns chat into tasks for leads, files the inbox, keeps the queue clean, distills decisions into the brain, posts one daily brief. |
| Area leads | Each owns an area of work, wakes once a day on a schedule, keeps its card and tasks current. |
| Dispatcher | A cheap poller that relays the owner's actions (decisions, comments, new tasks, captures) to the right agent within about a minute. |

## Data model

All documents share one versioned store: `{id, data, version, updatedAt, updatedBy}`. Writes are
optimistic: `PATCH` merges, `PUT` replaces, and both pin `if_version` (409 with the current version if it
moved). `{"__delete__": true}` removes a field.

### Brain
Free Markdown text holding the owner's preferences and standing judgment, with a full version history.
Each edit carries a `note` saying what changed. Every agent reads it at the start of every run.

### Items (the queue)
`items/<lead>-<yyyymmdd>-<slug>`, plus a short ref `Q-<n>` assigned by the server.

| Field | Meaning |
|---|---|
| `lead`, `area` | Who posted it, and what it is about |
| `kind` | `decision`, `review`, `question`, `deploy`, `brief` |
| `priority` | 1 blocks progress or touches production, data or money; 2 decide this week; 3 idea |
| `status` | `open` (Now), `later`, `approved`, `rejected`, `answered`, `done` |
| `title`, `summary`, `recommendation`, `options[]` | The ask. Summary and recommendation are sanitized Markdown |
| `links[]`, `media[]` | Links, and screenshots stored privately (media can be superseded, never deleted) |
| `choice`, `response`, `decidedAt`, `decidedBy` | The owner's decision; the server stamps `decidedAt` on every close |
| `ackedAt`, `outcome` | The lead acknowledged it, and what it did |
| `laterLog[]`, `laterUntil` | Moves between Now and Later; an optional day it comes back by itself |
| `changedAt`, `changeNote` | A lead changed an open item's content, and why (shown as **Updated**) |
| `deploy`, `approval`, `run`, `runs[]` | For deploy items: what to run, who approved it, and how the run went |

**Now and Later.** Now is the open items: the badge, notifications and "inbox zero" count only these.
Later is what the owner set aside, grouped by area, with no badge, optionally until a day.

### Tasks (the work)
`tasks/<lead>-<slug>`, short ref `T-<n>`. One per piece of work, or `kind: "question"`.

| Field | Meaning |
|---|---|
| `status` | `ready`, `active`, `blocked`, `waiting-owner`, `done` |
| `waitingOn` | Required when blocked: `{on: date, until}`, `{on: task, tasks[]}`, `{on: external, note}`, `{on: cases}`. Waiting on the owner is `waiting-owner`: Now (with an open `itemId`) or Later (`when: later`, optional `until`) |
| `priority`, `size`, `due`, `labels[]`, `area`, `body`, `links[]` | Planning fields |
| Question fields | `options[]`, `instances[{date, ref, note}]`, `decideWhen`, `decideAfter`, `leaning` |

The server keeps every change to title, status, priority, size, due, waits, labels and area as task events.
Date and task waits **unblock themselves**: a cron moves the task to `ready` when the wait is over.
Tasks have a comment thread shared by the owner and the lead.

### Areas and the inbox
`areas/<id>`: `title`, `memo` (where it stands, decisions, open questions), `leadId`, `links[]`, `order`,
plus an append-only **log** of the owner's raw thoughts. Everything the owner captures enters one
**inbox**. The chief of staff files each entry into an area, which writes the log in the same step, or
dismisses it.

### Lead cards
`leads/<id>`: name, focus, `plan`, `lastRun`, `lastSummary`, session link. The place for status.

## API

JSON over HTTPS under `/api`. The owner authenticates through the hosting provider's access layer;
agents use revocable bearer tokens with scopes (`read`, `items`, `leads`, `brain`, `admin`, and a
callback-only scope for the deploy workflow).

| Area | Endpoints |
|---|---|
| Brain | `GET /brain`, `PUT /brain {text, if_version, note}`, `GET /brain/history` |
| Documents | `GET/PUT/PATCH /items/<id>`, `/tasks/<id>`, `/leads/<id>`, `/areas/<id>`, `/views/<id>`; list with filters; `GET /items/Q-142`, `/tasks/T-88` by short ref |
| Media | `PUT /items/<id>/media?caption=…[&replace=1]`, `PATCH /items/<id>/media {supersede: [url]}`, `GET /media/<key>` |
| Deploys | `POST /items/<id>/deploy` (owner), `POST /items/<id>/deploy-chat-approval` (chief of staff, with the owner's exact words), `POST /items/<id>/deploy-result` (workflow callback) |
| Tasks | `GET /tasks/<id>/events`, `GET /task-events`, `GET/POST /tasks/<id>/comments`, `GET /comments` |
| Areas, inbox | `GET/POST /areas/<id>/log`, `GET/POST /inbox`, `POST /inbox/<id>/file|dismiss` |
| Queue | `GET/PUT /queue/busy {minutes, note}` (a self-expiring "hold approvals a moment" banner) |
| Jobs | `GET /jobs`, `POST /jobs/<id>/run` (owner): the scheduled jobs and a manual run |
| Admin | tokens, audit log, `GET /export`, `POST /import` (full round trip), push subscriptions |
| Ops | `GET /ops/pickup_poll.py`: the dispatcher's poller, served so it never runs a stale copy |

## Scheduled jobs

One minute cron, three jobs, each with its last run recorded and a **Run now** for the owner (same code
path): deploy notifications (sent ~2 minutes after a deploy item is posted, unless already decided),
auto-unblock (date and task waits, and dated Later tasks), and Later returns (dated Later items back to
Now, with a notification).

## The dispatcher

A poller that costs no model tokens while it waits. Every minute it compares the store with what it has
already relayed, and wakes the right agent once per event: the owner decided an item, a deploy run
finished (after a 90-second grace, so a lead watching its own run acks first), the owner added a task or
commented, a wait ended, a question collected the cases it needed, the owner captured something, an item
entered Now or a lead changed one (for the chief of staff's cross-item pass), or the owner pressed Start
on a parked task.

## Agent protocol

- **Every run:** read the brain, act on decided items and acknowledge them, keep tasks and the card
  current, and stop only when nothing is actionable (a stop hook enforces this).
- **Items:** lead with one bold line (the ask); short paragraphs and bullets; a recommendation and 2-4
  options; desktop and phone screenshots for product changes. Bundle steps that ship together. Say how an
  item relates to other open items (hard "approve after X", soft "better after X"). Check open items
  before posting; update instead of duplicating; send a `changeNote` when changing an open item;
  supersede old screenshots instead of reposting.
- **Decision tiers:** when the owner is undecided, log a question task with options and cases instead of
  guessing; when it collects enough cases, post one synthesis item.
- **Experiments:** provisional changes get a task that waits on a review date.
- **Refactors:** small debt is fixed in passing; bigger debt becomes a `refactor` task that a lead picks
  up when it has no ready work.

## Deploys

A deploy item names `{repo, workflow, ref, inputs: {branch, sha, …}}`. Only allowlisted
`repo:workflow@ref` pairs can run. On approval, the server dispatches the workflow. The workflow merges
exactly the approved commit from the latest main (it refuses if the branch moved), runs the
pending migrations it was told to expect, ships, and reports the run back to the item. A deploy item ships
exactly one variant; a note typed at approval can't change what ships.

**Approval paths:** in the app (owner only), or in chat with the chief of staff, which records the
owner's exact words. **Small-change lane:** small UI or docs changes the owner explicitly asked for may
be dispatched by the chief of staff without a separate approval. **Hand merges:** changes to workflow
files go to the owner as a pull request.

## Stack

A single serverless worker serves the API and the embedded single-page app; a SQLite-compatible database
holds documents and logs; an object store holds screenshots; web push notifies the owner's devices. The
SPA is installable as a home-screen app, refreshes on resume and picks up new deploys by itself.
