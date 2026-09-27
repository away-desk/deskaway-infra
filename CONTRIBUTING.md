# Contributing

- **DeskAway is in early development and is not accepting outside
  contributions yet.** Nothing here works; there is not yet enough in place
  for a contribution to sit on top of. Issues and questions are welcome.
- **Open an issue before a pull request.** Agreeing the shape of a change
  first is cheaper than reviewing the wrong one.
- **Branch off `main`, one unit of work per pull request, no direct pushes to
  `main`.** Fill in the pull request template rather than deleting it.
- **Run the formatter and linter before pushing:**
  ```sh
  terraform fmt -recursive
  checkov -d .        # config in policies/checkov
  ```
  `fmt` works but has nothing to format; checkov is not configured yet.
- **Add a `CHANGELOG.md` entry** under `## [Unreleased]` for anything
  notable.

Repo layout and conventions are in [AGENT.md](./AGENT.md). Licensed under
Apache-2.0; by contributing you agree your work is licensed the same way.
