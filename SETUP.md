# cowork-tools — New Machine Setup

Plugins and skills for [Claude Cowork](https://support.claude.com/en/articles/13345190-get-started-with-claude-cowork). No install — these are installed inside Claude Cowork via its plugin system.

## Clone (to edit / install locally)

```bash
git clone https://github.com/marcusk639/cowork-tools.git
cd cowork-tools
```

## Install plugins into Claude Cowork

From within a Claude Cowork session:

In the Claude desktop app, open **Customize → Plugins → Add marketplace** and enter
`marcusk639/cowork-tools`, then install `deep-research`. See `README.md` for details.

## Contents

| Directory                                | Plugin / Skill  | Description                                                      |
| ---------------------------------------- | --------------- | ---------------------------------------------------------------- |
| `.claude-plugin/`                        | (marketplace)   | Marketplace manifest listing the plugins                         |
| `deep-research/`                         | `deep-research` | Multi-source research; Firecrawl/Exa if enabled, else built-ins  |
| `prompt-engineer/`                       | `prompt-engineer` | Prompt-writing and prompt-debugging for Claude                 |

## Notes

- No `npm install` or build step required
- These files are configuration/prompt files consumed by Claude Cowork
- See `README.md` for full plugin comparison and setup instructions
