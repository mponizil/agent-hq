# Changelog

Platform changes, newest first. One entry per shipped change; instance-only changes (names, domains,
allowlist entries, brain edits) don't go here. See `SPEC.md` for the current shape of everything below.

## 2026-11-24
- **Spec: how the API passes the access layer.** One path-scoped bypass for `/api`; the server authenticates every
  request itself (owner identity, agent tokens, the deploy callback token, signed webhooks), so new outside callers
  need a credential, not another bypass.
- **Later opens on its board again.** The explicit-selection change pinned the first item in every view;
  Later now skips it, so its by-area board shows until you pick an item.
- **The chief of staff acts on the public repo as the app.** Comments, closing issues, docs sync PRs and merging
  a docs PR the owner approved go through the server as the code-host app; the merge itself checks the approval,
  the commit and that only root Markdown changes.
- **Safe server renames.** The pre-deploy step refuses to deploy a server name the hosting account doesn't have
  yet, since that would create a second server without its secrets: rename in the dashboard first, then deploy.
- **One address for agents.** Agents, the deploy callback and the app webhook use the owner's own domain at
  `/api` (Access is bypassed there; the server checks the token), so the server's platform name can change freely.
- **Dispatcher: ack after relay.** With `--wait-ack`, a hit counts as seen only after the dispatcher confirms it
  relayed it (`--ack <key>`); an unconfirmed hit is reported again after 10 minutes instead of being lost.
  A new test pairs each owner action (a hit) with the same action by an agent (no hit).
- **Code-host webhook and app.** The chief of staff hears about new commits, issues, pull requests and comments
  on the watched repos through the code host's signed app webhook, relayed by the dispatcher. The server's own
  calls (workflow dispatch, the lane's diff read) go out as the app when it's installed, so they're attributed to
  it; `GET /github/status` reports which credentials are set (yes/no only).
- **Hand-merge items say what the owner did:** their options read "Merged PR #N" and "Hold", never an
  action that sounds like the agent's.
- **The Queue never switches items under you.** On desktop the selected item is always in the URL, so a new,
  higher-priority item only joins the list; a typed reply or picked option stays with its own item.
- **Board rule on names:** the no-names rule covers other people. An owner who takes part openly may appear
  under their own name and their agents' names; agents never decide that on their own.
- **Replies are never lost.** Every reply the owner gives on an item is kept in `responses[]` and shown
  above the decision, so a second decision no longer erases the first.
- **Reopening counts as Updated.** A lead reopening a decided item with new text stamps `changedAt`, so the
  owner sees what changed.
- **Decided items keep the text that was decided:** after a decision, agents can't change an item's title,
  summary, recommendation or options (409). Outcome, ack and links stay writable; reopening is the way to re-ask.
- **Repo renames don't break deploys in flight:** `REPO_RENAMES` ("old=new") lets a run report back under
  the repo's new name for items posted before the rename.
- **The public board is GitHub Issues**, labelled `changelog`, `idea` and `question`: one issue per thread,
  replies as comments, posted directly under the rules in `CHANNEL.md`.
- **Leads read the owner's response before acting** on a decided item, and never call an item eligible
  for the small-change lane; `CLAUDE.md` changes never go through it.
- **Small-change lane, checked by the Worker.** The chief of staff ships through
  `POST /items/<id>/lane-dispatch`; the Worker reads the branch's diff and dispatches only UI and docs
  changes with no migrations. Anything else stays open for the owner.
- **Deploy items ship exactly one variant.** The approve dialog warns that a typed note can't change
  what ships, and offers to send it as a reply instead.
- **Screenshot revisions supersede instead of piling up.** Upload with `replace=1` or `PATCH …/media
  {supersede}`; the owner sees only current screenshots, with earlier ones folded away. Nothing is deleted.
- **Scheduled jobs registry with Run now.** Deploy notifications, auto-unblock and Later returns each
  record their last run; the owner can run one on demand from Settings.
- **Dispatcher grace period.** Run-completed and waiting-owner pings wait 90 seconds, so a lead that is
  already watching acks before it is pinged.
- **Quieter sync label.** The "synced" indicator only speaks up when data is actually stale.

## 2026-11-23
- **Fresh on resume.** The installed app refetches when it comes back to the foreground and shows a bar
  when a newer build is live.
- **Queue signals.** **Updated** badges with the lead's `changeNote` when an open item changes after the
  owner saw it; a self-expiring "queue updating" banner (`PUT /queue/busy`).
- **Short refs.** Every item and task gets `Q-n` / `T-n`; readable at `/api/items/Q-n`, linkable at `/q/n`.
- **Ordering across items.** Protocol: bundle steps that ship together, state hard and soft dependencies,
  check open items before posting, test-merge parallel deploy branches before calling them order-free.
- **Decided timestamps.** The server stamps `decidedAt` / `decidedBy` on every close; Decided sorts by it.
- **Tasks waiting on Later items** read "Later: yours" instead of a stale "Needs lead".

## 2026-11-22
- **Screenshot lightbox** with swipe, drag and tap-to-step on phones.

## 2026-11-21
- **Now and Later** replace Snooze. Later is grouped by area, has no badge, and can return on a chosen day.
- **Areas and the inbox.** Areas with a memo, links and an append-only log; one capture inbox filed by the
  chief of staff.
- Dark mode: softer borders and off-white text.
- Removed the legacy compatibility shims.

## 2026-11-18
- **Backup round trip.** `GET /export` and `POST /import` restore everything, documented and tested.
- Retired the original artifact-based queue page; the app is the only surface.

## 2026-11-17
- **Deploys merge onto the latest main**, not the branch's base, and refuse if the branch moved.
- Tasks use one Markdown `body` field.

## 2026-11-15
- **Hand merges as pull requests** for changes the deploy workflow can't push (workflow files).
- The dispatcher's poller is served by the API, so it never runs a stale copy.

## 2026-11-14
- **Tasks**: statuses, waits that unblock themselves, comments, events, and **questions** that collect cases.
- **Work** view: every lead's tasks in one board. Settings in tabs, a sectioned brain editor, full-width
  layout, pickup order for the dispatcher.

## 2026-11-12
- **Work board and backlog** with staleness flags; `blocked` (external) separated from `waiting-owner`.
- Item text renders as sanitized Markdown.
- Deploy race fixes; deploy hooks; screenshots stored in object storage; queue returns.
- Operations runbook; custom domain support.

## 2026-11-11
- **Initial release.** Worker + SQL API with versioned documents, brain history, lead cards and the
  queue app.
- Deploy approvals through an allowlisted GitHub Actions workflow, and the app deploying itself.
- Shared agent protocol, web push notifications, parallel worker sessions, the component-based UI.
