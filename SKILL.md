---
name: hermes-skills-lazy-index
description: Suppress the eager-skills-index from the Hermes system prompt. Adds a config gate so the full 2,147+ skill catalog never burns into context — skill discovery routes through vault_skills_vault_skill_lookup on demand instead.
triggers:
  - skills index context overflow
  - too many skills in system prompt
  - lazy load skills hermes
  - include_index_in_prompt
---

# Hermes Skills Lazy Index

## Problem

`build_skills_system_prompt()` in `agent/prompt_builder.py` eagerly builds an `<available_skills>` block containing every skill's name + description. At 2,147 skills this is ~100 KB per session — paid as tokens on every single API call regardless of whether those skills are needed.

## Solution

Two-line config change + one code patch. Adds a `skills.include_index_in_prompt` flag that skips `build_skills_system_prompt()` entirely when `false`. Skill discovery falls back to `vault_skills_vault_skill_lookup(query)` on demand.

## Changes

### 1. `~/.hermes/config.yaml`

```yaml
skills:
  external_dirs: []
  creation_nudge_interval: 15
  include_index_in_prompt: false   # ← ADD THIS LINE
```

### 2. `run_agent.py` — store config on AIAgent

In `AIAgent.__init__`, after loading the config (around line 1043):

```python
# Load config once for memory, skills, and compression sections
try:
    from hermes_cli.config import load_config as _load_agent_config
    _agent_cfg = _load_agent_config()
except Exception:
    _agent_cfg = {}
self._agent_cfg = _agent_cfg  # stored for use in _build_system_prompt
```

### 3. `run_agent.py` — gate the skills prompt

In `_build_system_prompt` (around line 2692), replace the skills section with:

```python
has_skills_tools = any(name in self.valid_tool_names for name in ['skills_list', 'skill_view', 'skill_manage'])
if has_skills_tools:
    # Respect config flag to suppress the full skills index from the system prompt.
    # When disabled, the agent uses vault_skills_vault_skill_lookup for on-demand
    # discovery instead of having all ~2100 skills burned into every session context.
    skills_config = self._agent_cfg.get("skills", {})
    include_index = skills_config.get("include_index_in_prompt", True)
    if include_index:
        avail_toolsets = {
            toolset
            for toolset in (
                get_toolset_for_tool(tool_name) for tool_name in self.valid_tool_names
            )
            if toolset
        }
        skills_prompt = build_skills_system_prompt(
            available_tools=self.valid_tool_names,
            available_toolsets=avail_toolsets,
        )
    else:
        skills_prompt = ""
else:
    skills_prompt = ""
if skills_prompt:
    prompt_parts.append(skills_prompt)
```

## Effect

| Skill count | Index in prompt | System prompt delta |
|-------------|-----------------|---------------------|
| 2,147 | 0 bytes | ~14.7K baseline |
| 9,000 | 0 bytes | ~14.7K baseline |
| 100,000 | 0 bytes | ~14.7K baseline |

Skill count becomes irrelevant. The index is only queried via `vault_skills_vault_skill_lookup(query)` or `vault_skill_recommend(task)` when the user actually needs a skill.

## Verification

```bash
# Check the snapshot size (should be empty/skipped when flag is false)
ls -la ~/.hermes/.skills_prompt_snapshot.json

# Restart session and watch token count on first prompt
# Should be ~14.7K instead of ~25K+
```

## Rollback

Set `include_index_in_prompt: true` in `config.yaml` and restart the session.
