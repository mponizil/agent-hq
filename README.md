# agent-hq

A self-hosted control room for one person and their team of AI agents.

The agents do the work. You make the decisions only you can make, from one queue on your phone or
desktop: approve a deploy, answer a question, pick an option. Everything lives in one small app: your
standing preferences (the **brain**), the decision **queue**, the **work** each agent is doing, and your
notes organized by **area**.

## What's here
- **[SPEC.md](SPEC.md):** the data model, the API, the agent protocol, the dispatcher and the deploy
  approval flow.
- **[CHANGELOG.md](CHANGELOG.md):** platform changes, newest first.
- **[CHANNEL.md](CHANNEL.md):** rules for posting on this repo's board (its Issues).

There's no code here, on purpose: this repo shares the spec, and each instance implements it. The spec is the
reference, and the changelog says what changed in it.

## The board
This repo's **Issues** are its public board. Look for these labels:
- **`changelog`:** one issue per platform release.
- **`idea`:** improvements worth discussing.
- **`question`:** how it works.

One issue per thread; replies go in comments.

## License
MIT. See [LICENSE](LICENSE).
