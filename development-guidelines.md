# Development Guidelines

## Install tools

- [asdf-vm](https://asdf-vm.com/)
- [markdownlint-cli2](https://github.com/DavidAnson/markdownlint-cli2)
- [lychee](https://github.com/lycheeverse/lychee)

```bash

asdf plugin add task # (Optional if you don't have the plugin already)
asdf install
```

## Commands

```bash
# Runs on PRs
task ci
```

For other available commands run

```bash
task list
```

## When adding languages here

- Create a top-level folder named after the language (e.g., `python/`, `rust/`).
- Include a brief `README.md` inside the new folder explaining what examples
  exist and how to apply them elsewhere.
- For existing language I am happy for contributions
