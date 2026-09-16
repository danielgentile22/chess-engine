# chess-engine

## Merge policy

This project overrides the global merge policy. Not everything needs a branch and a pull request.

**Commit straight to `main`:**

- Research notes and anything else under `docs/`
- Planning work: the wayfinder map, tickets, resolutions, roadmap edits
- README, licence, config, tooling scaffolding
- Small fixes and typos

**Open a pull request for review:**

- Engine code: move generation, search, evaluation, UCI
- Trainer and data-pipeline code
- Anything that changes how the engine plays, and so needs self-play results attached to the review
- Any change large enough that reading the diff in one sitting is the point

When in doubt, commit to `main`. A revert is cheap and this is a solo repo. The pull request is for changes where the review itself carries value, not as a gate on every commit.
