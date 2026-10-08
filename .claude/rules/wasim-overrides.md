# Wasim's fork overrides

This is Wasim's fork of cmux. Where this file and the upstream `CLAUDE.md` disagree, this file wins.

- Wasim's global rules govern the change loop, reviews, models and verification.
- The upstream `$autoreview` handoff and the dogfood and explicit-approval merge gate in `CLAUDE.md` ("First pass, then dogfood") don't apply here. Workers fix review findings, reviews follow the global size-scaled rule, and merges follow the global loop.
- Releases are not cut from this fork. Ignore the upstream Release section of `CLAUDE.md`: release secrets, `gh run watch --repo manaflow-ai/cmux` and `/release`.
- Keep upstream's test-first regression policy and its tagged-build and reload rules.
- Tagged builds and their cleanup are a shared resource: run one live check at a time.
