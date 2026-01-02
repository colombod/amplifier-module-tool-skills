# Skills Visibility Implementation Plan

**Date**: 2026-01-02
**Goal**: Make available skills automatically visible to agents
**Approach**: Integrated hook within tool-skills module

---

## Problem Statement

Currently, agents don't know what skills are available until they call `load_skill(list=true)`. This requires:
- Trial-and-error discovery
- Extra tool calls and API round-trips
- Agent guessing that skills might exist

**What other AI systems do**: Show available knowledge/skills in context automatically (Claude Projects, ChatGPT Custom GPTs, etc.)

## Exploration Findings

### How Tools Are Made Available

**Key Discovery**: Tools are passed as **structured data** in ChatRequest, NOT listed in system instructions text.

```python
# From amplifier-module-loop-basic/__init__.py:136-143
tools_list = [
    ToolSpec(name=t.name, description=t.description, parameters=t.input_schema)
    for t in tools.values()
]
chat_request = ChatRequest(messages=messages_objects, tools=tools_list)
```

**But**: This doesn't mean tools are invisible as text. Let's check foundation's actual pattern...

### Foundation's Pattern: Context-Appropriate Listing

**Discovery from foundation exploration:**

1. **Specialized agents LIST their tools** in markdown:
   - `agents/file-ops.md:41-47` - Lists read_file, write_file, edit_file, glob, grep
   - `agents/web-research.md:39-42` - Lists web_search, web_fetch
   - `agents/git-ops.md:37-39` - Lists bash tool

2. **Root bundle mentions high-level capabilities**:
   - `bundle.md:107` - "You have access to the **recipes** tool..."
   - Documents key tools, not every primitive

3. **Behaviors don't duplicate in text**:
   - YAML declares tools
   - No redundant markdown listing

**Pattern**: Agents that USE tools list them. Bundle composition handles provisioning.

### Existing Context Injection Mechanism

**HookResult with `inject_context` action** - established pattern from `hooks-todo-reminder`:

```python
HookResult(
    action="inject_context",
    context_injection="Text to inject",
    context_injection_role="user",
    ephemeral=True,  # Don't persist in history
    append_to_last_tool_result=True,
    suppress_output=True
)
```

**Events available**:
- `session:start` - Once at session initialization
- `provider:request` - Before each LLM call (used by todo-reminder)

**Precedent**: Foundation includes `hooks-todo-display` in `modules/hooks-todo-display/` (embedded hook, not separate repo)

---

## Architecture Decision

### ✅ Integrate Hook INTO tool-skills Module

**Why:**
- Skills discovery is tightly coupled to visibility
- Hook reuses `SkillsTool.skills` data (no duplication)
- One behavior declaration gets both capabilities
- Precedent: `hooks-todo-display` embedded in foundation
- Simpler for users: one module to install/configure

**Rejected**: Separate hooks-skills-context repo
- Would duplicate discovery logic or require complex coordination
- Two repos to version/maintain
- Breaks "skills as unit" philosophy

### Hook Design

**When**: `provider:request` event (before each LLM call)
**What**: Skills list formatted as markdown
**Format**: Name + description only (~30-50 tokens/skill)
**Cost**: ~500 tokens for 10 skills (acceptable, saves discovery round-trips)

---

## Implementation Specification

### File Changes Required

| File | Change | LOC Estimate |
|------|--------|--------------|
| `amplifier_module_tool_skills/hooks.py` | **CREATE** | ~80 lines |
| `amplifier_module_tool_skills/__init__.py` | **MODIFY** | +10 lines |
| `behaviors/skills.yaml` | **MODIFY** | +8 lines (config) |
| `bundle.md` | **MODIFY** | +30 lines (docs) |
| `tests/test_hooks.py` | **CREATE** | ~100 lines |

**Total new code**: ~220 lines (stays ruthlessly simple)

### 1. Create `amplifier_module_tool_skills/hooks.py`

```python
"""Skills visibility hook - makes available skills visible to agents."""

from typing import Any
from amplifier_core import HookResult

class SkillsVisibilityHook:
    """Hook that injects available skills list into context."""
    
    def __init__(self, skills: dict[str, Any], config: dict[str, Any]):
        """Initialize hook with skills data from tool.
        
        Args:
            skills: Dictionary of discovered skills from SkillsTool
            config: Hook configuration from visibility section
        """
        self.skills = skills
        self.enabled = config.get("enabled", True)
        self.inject_role = config.get("inject_role", "user")
        self.max_visible = config.get("max_skills_visible", 50)
        self.ephemeral = config.get("ephemeral", True)
        self.priority = config.get("priority", 20)
    
    async def on_provider_request(self, event: str, data: dict[str, Any]) -> HookResult:
        """Inject skills list before LLM request.
        
        Event: provider:request
        Timing: Before each LLM call
        """
        if not self.enabled or not self.skills:
            return HookResult(action="continue")
        
        # Build skills list
        skills_text = self._format_skills_list()
        
        # Inject as context
        return HookResult(
            action="inject_context",
            context_injection=skills_text,
            context_injection_role=self.inject_role,
            ephemeral=self.ephemeral,
            append_to_last_tool_result=True,
            suppress_output=True
        )
    
    def _format_skills_list(self) -> str:
        """Format skills list for injection."""
        if not self.skills:
            return ""
        
        # Limit to max_visible
        skills_items = sorted(self.skills.items())[:self.max_visible]
        
        lines = ["<available-skills>", "Available skills (use load_skill tool):"]
        for name, metadata in skills_items:
            lines.append(f"- **{name}**: {metadata.description}")
        
        if len(self.skills) > self.max_visible:
            remaining = len(self.skills) - self.max_visible
            lines.append(f"_({remaining} more available via load_skill)_")
        
        lines.append("</available-skills>")
        return "\n".join(lines)
```

### 2. Update `amplifier_module_tool_skills/__init__.py`

Add after tool mounting (around line 46):

```python
# Mount skills visibility hook if enabled
visibility_config = config.get("visibility", {})
if visibility_config.get("enabled", True):  # Default: enabled
    from amplifier_module_tool_skills.hooks import SkillsVisibilityHook
    
    hook = SkillsVisibilityHook(tool.skills, visibility_config)
    
    # Register hook on provider:request event
    unregister = coordinator.hooks.register(
        event="provider:request",
        handler=hook.on_provider_request,
        priority=hook.priority,
        name="skills-visibility"
    )
    
    logger.info(f"Mounted skills visibility hook with {len(tool.skills)} skills")
```

### 3. Update `behaviors/skills.yaml`

```yaml
bundle:
  name: skills-behavior
  version: 1.0.0
  description: Anthropic Skills support with progressive disclosure and automatic visibility

tools:
  - module: tool-skills
    source: git+https://github.com/microsoft/amplifier-module-tool-skills@main
    config:
      # Default skills directories
      # skills_dirs: []  # Uses defaults: .amplifier/skills, ~/.amplifier/skills
      
      # Skills visibility (automatic context injection)
      visibility:
        enabled: true              # Show available skills to agent
        inject_role: "user"        # Inject as user message
        max_skills_visible: 50     # Limit for large skill sets
        ephemeral: true            # Don't persist in history (lightweight)
        priority: 20               # Hook priority (after todo:10, before user:50+)
```

### 4. Update `bundle.md`

Add new section after "Available Tool" (around line 35):

```markdown
## Skills Visibility

When this bundle is included, agents **automatically see** a list of available skills in their context before each request. This enables:

- **Immediate Discovery**: Agents know what domain knowledge exists without guessing
- **Progressive Disclosure**: Only metadata (name + description) is shown initially
- **On-Demand Loading**: Full content loaded only when needed via `load_skill()`

**What agents see:**

```
<available-skills>
Available skills (use load_skill tool):
- **python-testing**: Best practices for Python testing with pytest
- **git-workflow**: Git branching and commit message standards
- **api-design**: RESTful API design patterns and conventions
</available-skills>
```

Skills are injected before each LLM request using an integrated hook, ensuring agents always have current skill awareness.

### Disable Visibility

If you prefer manual discovery only:

```yaml
tools:
  - module: tool-skills
    config:
      visibility:
        enabled: false
```
```

### 5. Create `tests/test_hooks.py`

```python
"""Tests for skills visibility hook."""

import pytest
from pathlib import Path
from amplifier_module_tool_skills.hooks import SkillsVisibilityHook
from amplifier_module_tool_skills.discovery import SkillMetadata

@pytest.fixture
def mock_skills():
    """Mock skills data."""
    return {
        "python-testing": SkillMetadata(
            name="python-testing",
            description="Best practices for Python testing",
            path=Path("/skills/python-testing/SKILL.md"),
            source="/skills",
        ),
        "git-workflow": SkillMetadata(
            name="git-workflow",
            description="Git branching standards",
            path=Path("/skills/git-workflow/SKILL.md"),
            source="/skills",
        ),
    }

@pytest.mark.asyncio
async def test_hook_injects_skills_list(mock_skills):
    """Test hook injects formatted skills list."""
    hook = SkillsVisibilityHook(mock_skills, {})
    result = await hook.on_provider_request("provider:request", {})
    
    assert result.action == "inject_context"
    assert "Available skills" in result.context_injection
    assert "python-testing" in result.context_injection
    assert "git-workflow" in result.context_injection

@pytest.mark.asyncio
async def test_hook_respects_enabled_flag(mock_skills):
    """Test hook can be disabled."""
    hook = SkillsVisibilityHook(mock_skills, {"enabled": False})
    result = await hook.on_provider_request("provider:request", {})
    
    assert result.action == "continue"

@pytest.mark.asyncio
async def test_hook_handles_empty_skills():
    """Test hook handles no skills gracefully."""
    hook = SkillsVisibilityHook({}, {})
    result = await hook.on_provider_request("provider:request", {})
    
    assert result.action == "continue"

@pytest.mark.asyncio
async def test_hook_limits_max_visible():
    """Test max_skills_visible configuration."""
    many_skills = {
        f"skill-{i}": SkillMetadata(
            name=f"skill-{i}",
            description=f"Skill {i}",
            path=Path(f"/skills/skill-{i}/SKILL.md"),
            source="/skills",
        )
        for i in range(100)
    }
    
    hook = SkillsVisibilityHook(many_skills, {"max_skills_visible": 10})
    result = await hook.on_provider_request("provider:request", {})
    
    assert "(90 more available" in result.context_injection
```

---

## Required Changes Summary

### Changes to tool-skills Module Only ✅

**No changes needed to:**
- ✅ amplifier-core (kernel)
- ✅ amplifier-foundation (bundle system)
- ✅ Any other repositories

**All changes in amplifier-module-tool-skills:**
1. Create `amplifier_module_tool_skills/hooks.py` (~80 lines)
2. Update `amplifier_module_tool_skills/__init__.py` (+10 lines)
3. Update `behaviors/skills.yaml` (+8 lines config)
4. Update `bundle.md` (+30 lines docs)
5. Create `tests/test_hooks.py` (~100 lines)

**Total**: ~228 lines of new code

---

## Token Cost Analysis

**Per skill**: ~30-50 tokens (name + description)
- Example: `- **python-testing**: Best practices for Python testing` ≈ 15 tokens

**Typical usage**:
- 10 skills = ~500 tokens per request
- 50 skills = ~2,500 tokens per request (still reasonable)
- 100+ skills = use max_visible limit

**Trade-off**: 
- Cost: +500 tokens per request (10 skills)
- Benefit: Saves 1-2 tool calls for discovery (200-500 tokens saved)
- Net: Roughly neutral, but better UX (immediate awareness)

---

## Configuration Options

```yaml
tools:
  - module: tool-skills
    config:
      # Skills directories
      skills_dirs:
        - ~/.amplifier/skills
      
      # Visibility configuration
      visibility:
        enabled: true              # Enable/disable automatic listing
        inject_role: "user"        # Message role for injection
        max_skills_visible: 50     # Limit displayed skills
        ephemeral: true            # Don't persist in history
        priority: 20               # Hook priority
```

---

## Implementation Steps

1. **Create hooks.py** - Skills visibility hook class
2. **Update __init__.py** - Mount hook after tool
3. **Update behaviors/skills.yaml** - Add visibility config
4. **Update bundle.md** - Document visibility feature
5. **Create tests** - Verify hook behavior
6. **Test end-to-end** - Verify skills appear in context

---

## Follow-Up Question

**Should this be in the current PR or a separate PR?**

**Option A: Add to current PR** (bundle-integration)
- All skills improvements together
- Single review cycle
- More changes but cohesive

**Option B: Separate PR** (skills-visibility)
- Keep bundle integration focused
- Visibility is enhancement, not requirement
- Easier to review separately

Which approach do you prefer?
