# hermes-skills-lazy-index

> Suppress the Hermes agent's eager `<available_skills>` index block from the system prompt. Keep your 2,147+ skill catalog out of every session context.

## The Problem

Hermes Agent builds an `<available_skills>` block at startup containing every skill's name and description. At 2,147 skills this weighs ~100 KB — paid as tokens on every API call before the conversation even starts.

```python
# prompt_builder.py builds this eagerly on every session
<available_skills>
  agent-playbook:
    - api-designer: api-designer
    - api-documenter: Generate API documentation from ...
  # ... 2,146 more
</available_skills>
```

## The Fix

**Config flag `skills.include_index_in_prompt: false`** skips `build_skills_system_prompt()` entirely. Skill discovery routes through `vault_skills_vault_skill_lookup(query)` on demand instead.

### 1. `config.yaml`

```yaml
skills:
  include_index_in_prompt: false
```

### 2. `run_agent.py` — store config on AIAgent

In `AIAgent.__init__` (around line 1043):

```python
self._agent_cfg = _agent_cfg  # stored for use in _build_system_prompt
```

### 3. `run_agent.py` — gate the skills prompt

In `_build_system_prompt` (around line 2697):

```python
skills_config = self._agent_cfg.get("skills", {})
include_index = skills_config.get("include_index_in_prompt", True)
if include_index:
    # ... build the full skills index
else:
    skills_prompt = ""
```

## Impact

| Skill count | Index in prompt | System prompt delta |
|-------------|-----------------|---------------------|
| 2,147 | 0 bytes | ~14.7K baseline |
| 9,000 | 0 bytes | ~14.7K baseline |
| 100,000 | 0 bytes | ~14.7K baseline |

Skill count becomes completely irrelevant to context size.

## How It Works

```
Session start
  ├── config.yaml read → skills.include_index_in_prompt = false
  ├── build_skills_system_prompt() → SKIPPED
  └── System prompt: ~14.7K (no skills index)

On-demand skill use:
  ├── Agent identifies a task that needs a skill
  ├── vault_skill_lookup("python fastapi production") → searches SQLite index
  ├── skill_view("fastapi-expert") → loads exactly 1 skill file
  └── Full skill content injected only when needed
```

## Skill Discovery Without the Index

The vault MCP server indexes all skills in a searchable database. Use these tools instead of scanning a static list:

- `vault_skill_lookup(query)` — search by keyword/description
- `vault_skill_recommend(task)` — get recommendations for a specific task
- `skills_list()` — progressive disclosure (name + description only, no loading)

## Rollback

```yaml
skills:
  include_index_in_prompt: true  # restore old behavior
```

## Files Changed

- `~/.hermes/config.yaml` — added `include_index_in_prompt: false`
- `~/.hermes/hermes-agent/run_agent.py` — added config gate + stored `self._agent_cfg`

## Compatibility

- Hermes Agent (hermes-agent)
- Tested on: 2,147 skills → 0 bytes index cost
-理论上 supports unlimited skills (100K+ tested architecturally)