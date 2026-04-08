# CLAUDE.md — browserless-tooling

Setup scripts for Playwright/Chromium-based tooling.

`human_browser` container workspace at `/docker/human_browser/workspace` is shared with `aichat-video`, `aichat-ocr`, and `aichat-pdf`.

## Build

```bash
bash aibrowse-setup.sh      # provision Browserless Chromium
bash browsewrap-setup.sh    # provision LM Studio MCP wrappers
```

## Test

```bash
bash -n aibrowse-setup.sh browsewrap-setup.sh   # syntax check
shellcheck aibrowse-setup.sh browsewrap-setup.sh
```

## Lint

```bash
shellcheck aibrowse-setup.sh browsewrap-setup.sh
```
