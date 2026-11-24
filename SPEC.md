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
| `responses[]` | Every reply the owner gave, oldest first (`{at, by, text, choice, status}`), appended by the server, so a second decision never erases the first |
| `ackedAt`, `outcome` | The lead acknowledged it, and what it did |
| `laterLog[]`, `laterUntil` | Moves between Now and Later; an optional day it comes back by itself |
| `changedAt`, `changeNote` | A lead changed an open item's content, or reopened a decided one with new text, and why (shown as **Updated**) |
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

**One access-layer exemption, for the whole API.** The owner's pages sit behind the hosting provider's access
layer (single sign-on for the owner only). `/api` is exempted from it as one path-scoped bypass, and the server
authenticates **every** `/api` request itself, so nothing reaches the store on the strength of the bypass alone:
- **The owner's app:** the server verifies the access layer's signed identity (its JWT, from the header or
  cookie it sets) against the owner's address, so the owner's browser still calls `/api` signed in.
- **Agents:** a revocable bearer token with scopes, stored only as a hash.
- **The deploy workflow:** a callback-only token that can report a run on an item and nothing else.
- **The code host's webhook:** its HMAC signature over the raw body, checked against a shared secret.

Every caller that can't sign in, such as CI, webhooks or other machines, is then a new credential type, never a new
bypass, and one address (the owner's domain plus `/api`) serves people and machines alike. Instances that keep
the access layer on the API instead need one bypass per outside caller, or service tokens.

**Agent channel: MCP (optional).** An instance may also serve an MCP server (`POST /mcp`, Streamable HTTP)
as the agents' channel: thin tools over the same store as the REST API (read the brain, post and ack items,
update tasks and cards, file the inbox), with the same `if_version` rules, and authenticated like any
agent. Agents then don't need to know URLs or payload shapes, and every write is attributed by the channel it
came through: writes from the owner's app are the owner's, writes through MCP or an agent token are an
agent's. The dispatcher relays only the owner's writes (plus time-based events), so an agent never wakes
itself or another agent with its own changes.

| Area | Endpoints |
|---|---|
| Brain | `GET /brain`, `PUT /brain {text, if_version, note}`, `GET /brain/history` |
| Documents | `GET/PUT/PATCH /items/<id>`, `/tasks/<id>`, `/leads/<id>`, `/areas/<id>`, `/views/<id>`; list with filters; `GET /items/Q-142`, `/tasks/T-88` by short ref |
| Media | `PUT /items/<id>/media?caption=…[&replace=1]`, `PATCH /items/<id>/media {supersede: [url]}`, `GET /media/<key>` |
| Deploys | `POST /items/<id>/deploy` (owner), `POST /items/<id>/deploy-chat-approval` (chief of staff, with the owner's exact words), `POST /items/<id>/lane-dispatch` (chief of staff, small-change lane), `POST /items/<id>/deploy-result` (workflow callback) |
| Tasks | `GET /tasks/<id>/events`, `GET /task-events`, `GET/POST /tasks/<id>/comments`, `GET /comments` |
| Areas, inbox | `GET/POST /areas/<id>/log`, `GET/POST /inbox`, `POST /inbox/<id>/file|dismiss` |
| Queue | `GET/PUT /queue/busy {minutes, note}` (a self-expiring "hold approvals a moment" banner) |
| Jobs | `GET /jobs`, `POST /jobs/<id>/run` (owner): the scheduled jobs and a manual run |
| Code host | `POST /hooks/github` (the code host's app webhook, signed, no token), `GET /repo-events?since=` (activity on the watched repos, kept 7 days), `GET /github/status` (which credentials are set, yes/no only, and whether the app authenticates) |
| Admin | tokens, audit log, `GET /export`, `POST /import` (full round trip), push subscriptions |
| Ops | `GET /ops/pickup_poll.py`: the dispatcher's poller, served so it never runs a stale copy |

**Decided items are frozen** for agents: once an item leaves Now and Later, its `title`, `summary`,
`recommendation` and `options` can't change (409). Everything else (outcome, ack, links, media, deploy
run) stays writable; reopening the item is the way to re-ask.

**Short refs** (`Q-<n>`, `T-<n>`) are one global sequence per collection rather than per-area prefixes: they
never collide and survive an item or task moving to another area.

**Items vs tasks:** an item is a moment where the owner decides; a task is a piece of work. In practice an
item works like "a task with an optional decision step"; whether to merge the two is an open question.

## Scheduled jobs

One minute cron, three jobs, each with its last run recorded and a **Run now** for the owner (same code
path): deploy notifications (sent ~2 minutes after a deploy item is posted, unless already decided),
auto-unblock (date and task waits, and dated Later tasks), and Later returns (dated Later items back to
Now, with a notification). Recording every run costs about 1,440 small writes a day, accepted for the
visibility.

## The dispatcher

A poller that costs no model tokens while it waits. Every minute it compares the store with what it has
already relayed, and wakes the right agent once per event: the owner decided an item, a deploy run
finished (after a 90-second grace, so a lead watching its own run acks first), the owner added a task or
commented, a wait ended, a question collected the cases it needed, the owner captured something, an item
entered Now or a lead changed one (for the chief of staff's cross-item pass), the owner pressed Start
on a parked task, or a watched repo had new activity (for the chief of staff, from the code host's webhook; what the
server's own app did, and the chief of staff's own pushes, are skipped).

**Ack after relay:** a hit counts as relayed only once the dispatcher confirms it (`--ack <key>`) after the
message went through. A hit that wasn't confirmed is reported again after a few minutes, so a failed relay (a busy
session, an errored turn, a restart) is retried instead of lost. Only the owner's actions and server-side events
raise hits; the same action by an agent raises none, so agents never wake themselves.

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
be dispatched by the chief of staff without a separate approval, through `POST /items/<id>/lane-dispatch`:
the server reads the branch's diff and dispatches only if every file is UI or docs (never the agents'
rule file) and no migrations ship; otherwise the item stays open for the owner. Why it's shaped this way:
- **The file check is the enforcement.** Who may call the lane is a self-declared header; what ships is
  decided by the diff, which the agent can't influence.
- **It fails closed:** a 403 or 404 from the code host, an empty file list, or a diff too large to list in
  full (100+ files) all refuse with 422.
- **Build config is deliberately outside** the allowlist (bundler, package and TypeScript config can change
  what ships), and a renamed file counts under both its old and new path.
- **Rejected:** a `lane` input on the deploy workflow (a workflow change needs a hand merge, and the input
  would be bypassable), and a per-option "this option deploys" flag on items (a deploy item ships exactly one
  variant instead).

**Repo renames:** `REPO_RENAMES` ("old=new") lets runs of items posted before a rename report under the new
name. It maps one way only, and should be emptied once the old name belongs to another repository.

**Acting on the code host:** the server's own calls (dispatching a workflow, reading a diff for the lane) go out
as the instance's code-host app when it's configured and installed on that repo, so they're attributed to the
app; otherwise, or if the app is refused, with a personal token. The chief of staff's own actions on the public
repo (comments, closing issues, docs sync PRs, merging a docs PR the owner approved) also go through the server
as the app: `/github/issues/<n>/comments`, `/github/issues/<n>`, `/github/sync-pr`, `/github/pulls/<n>/merge`. The
merge checks that the owner approved an item linking that PR, that the PR is at the approved commit, and that
it only touches root-level Markdown.

**Hand merges:** changes to workflow files go to the owner as a pull request. The item's options say what
the owner did ("Merged PR #N", "Hold"), never an instruction to an agent: an option reading "Merge PR #N"
was once taken to mean an agent would merge it. An instance may let the chief of staff merge docs-only PRs (for
example, syncing the public docs) once the owner approves the item; then the options are "Approve" / "Hold"
and the approval is the merge.

## Stack

A single serverless worker serves the API and the embedded single-page app; a SQLite-compatible database
holds documents and logs; an object store holds screenshots; web push notifies the owner's devices. The
SPA is installable as a home-screen app, refreshes on resume and picks up new deploys by itself.
