# sciyth-skills

Sciyth Studio's shared Claude Code skills, published as a plugin marketplace. One copy serves every game, so a fix lands everywhere.

## Plugins

- **roblox-dev**: `roblox-gui`, `roblox-input`, `roblox-monetization`, `roblox-studio-mcp`, `studio-playtest`, `studio-ui`, `vfx`, `model-workflow` (includes the Blender kit and meshlab), `batch-model`, `studify`, `item-icons`.

## Using it in a game repo

Add this to the game's `.claude/settings.json` (committed), so everyone who opens the repo is offered the plugin:

```json
{
  "extraKnownMarketplaces": {
    "sciyth-skills": { "source": { "source": "github", "repo": "Sciyth-Studio/sciyth-skills" } }
  },
  "enabledPlugins": { "roblox-dev@sciyth-skills": true }
}
```

Or install it by hand in Claude Code: `/plugin marketplace add Sciyth-Studio/sciyth-skills`, then `/plugin install roblox-dev@sciyth-skills`.

Skills from a plugin show up namespaced, e.g. `roblox-dev:vfx`. Game-specific skills (a game's own rocks, zones or bosses) stay in that game's `.claude/skills/`.

## Changing a skill

Edit it here, bump `version` in `plugins/roblox-dev/.claude-plugin/plugin.json`, commit and push. Games pick it up with `/plugin marketplace update sciyth-skills`.

Scripts in the skills run from the game's root folder and read and write that game's `assets/`.
