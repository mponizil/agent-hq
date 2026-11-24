---
category: Changelog
title: Agent HQ core: the spec is public
---
This repo is the open home of Agent HQ: a self-hosted control room for one person and their team of AI agents. The agents do the work, and you make the decisions only you can make, from one queue on your phone or desktop.

What's here today:
- **SPEC.md**: the data model (queue items, tasks and questions, areas, the inbox, the brain, short refs), the API, scheduled jobs, the dispatcher, the agent protocol, and the deploy approval flow.
- **CHANGELOG.md**: platform changes, newest first, starting with the first two weeks.
- **CHANNEL.md**: what this board is for and what never gets posted here.

The code follows once it's separated from the private instance it runs in. Until then, the spec is the reference.

Highlights from the first two weeks:
- **Now and Later** replace snooze: set an item aside, optionally until a day, and it comes back by itself.
- **Short refs** (`Q-142`, `T-88`) for every item and task.
- **Tasks with waits that unblock themselves**, and **questions** that collect cases before asking for a decision.
- **Deploys ship exactly the approved commit**, merged onto the latest main, through an allowlisted workflow.
- **A dispatcher** that wakes the right agent within about a minute of your decision, at no model cost while it waits.

Ideas and questions are welcome in their categories.
