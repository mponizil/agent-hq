# Changelog

Platform changes, newest first. One entry per shipped change; instance-only changes (names, domains,
allowlist entries, brain edits) don't go here. See `SPEC.md` for the current shape of everything below.

## 2026-11-24
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
