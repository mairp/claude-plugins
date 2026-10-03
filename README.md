# mairp's Claude Code plugins

A [Claude Code](https://claude.com/claude-code) plugin marketplace — practical
tools for AI infrastructure, security, and agent operations.

## Use it

```
/plugin marketplace add mairp/claude-plugins
/plugin install <plugin>@mairp
```

Then `/plugin` to browse, enable, or disable what you installed.

## Plugins

| Plugin | What it does |
|---|---|
| [`antares-scan`](https://github.com/mairp/antares-scan) | Fast, free, **first-pass** security smell test. Points a tiny local LLM (antares-1b, a ~2 GB Granite-4.0 fine-tune) at a folder for cheap vulnerability leads across a codebase in seconds — a triage aid before a deeper audit, **not** a substitute for Semgrep, CodeQL, or Bandit. |
| [`mixture-of-loops`](https://github.com/mairp/mixture-of-loops) | Derives a provenance-bound launch contract and an unattended Specstride pipeline from GitHub Spec Kit artifacts. Sources are pinned by SHA-256, so editing a spec after derivation blocks the launch until the contract is regenerated. |
| [`speckit-batch`](plugins/speckit-batch) | Runs a Spec Kit command over a range of specs on a chosen harness, with the model and reasoning effort pinned per run, sequentially or several features in parallel. |
| [`video-to-deck`](https://github.com/mairp/video-to-deck) | Turns videos into Marp slide decks (PPTX, PDF, HTML) with local Whisper, OCR and a fresh agent per video. |
| [`qmd-recall`](https://github.com/mairp/qmd-gateway) | Recall from and write to a shared qmd memory gateway, so every agent in a fleet uses the same local RAG memory. |
| [`gpu-ops`](https://github.com/mairp/gpu_rtx_3090) | Operates an RTX 3090 Thunderbolt eGPU and the local models served on it: status, safe drain and power-off, power-up, model load checks. |

Install a single plugin, e.g.:

```
/plugin marketplace add mairp/claude-plugins
/plugin install antares-scan@mairp
```

## License

Each plugin carries its own license (see its repository). This marketplace
catalog is Apache-2.0.
