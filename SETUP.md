# cowork-tools — New Machine Setup

Plugins and skills for [Claude Cowork](https://support.claude.com/en/articles/13345190-get-started-with-claude-cowork). No install — these are installed inside Claude Cowork via its plugin system.

## Clone (to edit / install locally)

```bash
git clone https://github.com/marcusk639/cowork-tools.git
cd cowork-tools
```

## Install plugins into Claude Cowork

From within a Claude Cowork session:

```
/plugin install path/to/deep-research-plugin
```

Or point Cowork to the GitHub repo URL when prompted for a plugin source.

## Contents

| Directory                                | Plugin / Skill  | Description                                                      |
| ---------------------------------------- | --------------- | ---------------------------------------------------------------- |
| `cowork-tools-deep-research-overlay/`    | `deep-research` | Multi-source research using Cowork built-in search (no API keys) |
| `cowork-tools-deep-research-overlay-v0/` | (legacy)        | Earlier version                                                  |
| `skills/`                                | Various         | Additional Cowork skills                                         |

## Notes

- No `npm install` or build step required
- These files are configuration/prompt files consumed by Claude Cowork
- See `README.md` for full plugin comparison and setup instructions
