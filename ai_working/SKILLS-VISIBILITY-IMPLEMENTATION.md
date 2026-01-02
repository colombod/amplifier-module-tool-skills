# Skills Visibility Implementation Plan

**Date**: 2026-01-02
**Status**: Ready for implementation

---

## The Core Problem

**Tools vs. Skills - Different Visibility Mechanisms:**

| Category | How Agents See Them | Mechanism |
|----------|-------------------|-----------|
| **Tools** (bash, read_file, etc.) | Structured data in ChatRequest.tools | LLM API's function calling |
| **Skills** (knowledge sources) | ❌ Currently: Only via load_skill(list=true) | Text context only |

**The Issue**: Skills are **knowledge sources**, not executable functions. They CAN'T be in ChatRequest.tools because they're not tools - they're content that the load_skill tool retrieves.

**What We Need**: Inject available skills list into text context, like hooks-todo-reminder does.

---

## Research Findings

### How Agents Know About Tools

**From session analysis:**
1. **Structured Data**: Tools sent in `ChatRequest.tools` parameter to Anthropic API
   - Contains: name, description, input_schema (JSON schema)
   - Example: 12 tools with rich descriptions (254-7500+ chars each)
   - This is how Claude knows "I can call read_file with these parameters"

2. **System Instructions**: Policies about tool usage, NOT tool catalogs
   - Contains: "use todo tool for complex tasks", "delegate to foundation:explorer when..."
   - Does NOT contain: comprehensive list of available tools
   - Assumes agent already knows tools (from structured data)

**The Pattern**: 
- Foundation trusts ChatRequest.tools to make tools visible
- System instructions focus on WHEN/HOW to use tools, not WHAT tools exist

### Why Skills Need Different Approach

**Skills are not tools** - they're knowledge that a tool (load_skill) retrieves. The load_skill tool appears in ChatRequest.tools, but individual skills don't.

**Other AI systems** (Claude Projects, ChatGPT) list available knowledge sources in context as text. We should do the same.

---

## Solution: Integrated Hook in tool-skills Module

### Architecture

**Single module provides both:**
1. `SkillsTool` - Discovers and loads skills (existing)
2. `SkillsVisibilityHook` - Injects skills list into context (new)

**Benefits:**
- ✅ No separate repo to maintain
- ✅ Hook reuses tool's skills discovery data
- ✅ One behavior declaration gets both capabilities
- ✅ Tight coupling (visibility IS part of skills feature)
- ✅ Precedent: `hooks-todo-display` embedded in foundation

### When to Inject

**Event**: `provider:request` (before each LLM call)

**Why not session:start?**
- Skills can be added/changed during session
- Per-turn injection ensures current state
- Ephemeral mode keeps history clean
- Follows hooks-todo-reminder pattern

---

## Implementation Specification

### File Structure

```
amplifier-module-tool-skills/
├── amplifier_module_tool_skills/
│   ├── __init__.py          # [MODIFY] Mount both tool and hook
│   ├── discovery.py         # [NO CHANGE]
│   └── hooks.py             # [CREATE] ~80 lines
├── behaviors/
│   └── skills.yaml          # [MODIFY] Add visibility config
├── bundle.md                # [MODIFY] Document visibility
└── tests/
    └── test_hooks.py        # [CREATE] ~100 lines
```

**Total new code**: ~220 lines

### 1. Create `amplifier_module_tool_skills/hooks.py`

```python
"""Skills visibility hook - makes available skills visible to agents."""

from typing import Any
from amplifier_core import HookResult

class SkillsVisibilityHook:
    """Hook that injects available skills list into context before each LLM call."""
    
    def __init__(self, skills: dict[str, Any], config: dict[str, Any]):
        """Initialize hook with skills data from tool.
        
        Args:
            skills: Dictionary of discovered skills (from SkillsTool.skills)
            config: Hook configuration from visibility section
        """
        self.skills = skills  # Reference to tool's skills dict
        self.enabled = config.get("enabled", True)
        self.inject_role = config.get("inject_role", "user")
        self.max_visible = config.get("max_skills_visible", 50)
        self.ephemeral = config.get("ephemeral", True)
    
    async def on_provider_request(self, event: str, data: dict[str, Any]) -> HookResult:
        """Inject skills list before LLM request.
        
        Event: provider:request (before each LLM call)
        """
        if not self.enabled or not self.skills:
            return HookResult(action="continue")
        
        skills_text = self._format_skills_list()
        
        return HookResult(
            action="inject_context",
            context_injection=skills_text,
            context_injection_role=self.inject_role,
            ephemeral=self.ephemeral,
            append_to_last_tool_result=True,
            suppress_output=True
        )
    
    def _format_skills_list(self) -> str:
        """Format skills list as markdown."""
        if not self.skills:
            return ""
        
        # Sort and limit
        skills_items = sorted(self.skills.items())[:self.max_visible]
        
        lines = ["<available-skills>"]
        lines.append("Available skills (load with load_skill tool):")
        
        for name, metadata in skills_items:
            lines.append(f"- **{name}**: {metadata.description}")
        
        # Show truncation if applicable
        if len(self.skills) > self.max_visible:
            remaining = len(self.skills) - self.max_visible
            lines.append(f"_({remaining} more - use load_skill(list=true) to see all)_")
        
        lines.append("</available-skills>")
        return "\n".join(lines)
```

**Key Design Points:**
- Shares `tool.skills` reference (no duplicate discovery)
- Default enabled (can be disabled)
- Uses `<available-skills>` tags for clarity
- Ephemeral by default (doesn't bloat history)
- Follows hooks-todo-reminder pattern exactly

### 2. Update `amplifier_module_tool_skills/__init__.py`

Add after tool mounting (after line 46):

```python
    # Mount tool
    await coordinator.mount("tools", tool, name=tool.name)
    
    # Mount skills visibility hook (NEW - add this section)
    visibility_config = config.get("visibility", {})
    if visibility_config.get("enabled", True):  # Default: enabled
        from amplifier_module_tool_skills.hooks import SkillsVisibilityHook
        
        hook = SkillsVisibilityHook(tool.skills, visibility_config)
        
        # Register on provider:request event
        coordinator.hooks.register(
            event="provider:request",
            handler=hook.on_provider_request,
            priority=visibility_config.get("priority", 20),
            name="skills-visibility"
        )
        
        logger.info(f"Mounted skills visibility hook ({len(tool.skills)} skills)")
    
    # Emit discovery event (existing code)
    await coordinator.hooks.emit("skills:discovered", {...})
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
      # Skills directories (default: .amplifier/skills, ~/.amplifier/skills)
      # skills_dirs:
      #   - /custom/path
      
      # Automatic skills visibility (injected before each LLM request)
      visibility:
        enabled: true              # Show available skills to agent
        inject_role: "user"        # Role for injected message
        max_skills_visible: 50     # Limit for large skill sets
        ephemeral: true            # Don't persist in history
        priority: 20               # Hook execution priority
```

**Note**: No separate hook declaration needed - tool module handles both

### 4. Update `bundle.md`

Add section after "Available Tool" (around line 35):

```markdown
## Automatic Skills Visibility

When this bundle is included, agents **automatically see** a list of available skills before each request. This enables discovery without trial-and-error.

**What agents see:**

```
<available-skills>
Available skills (load with load_skill tool):
- **python-testing**: Best practices for Python testing with pytest
- **git-workflow**: Git branching and commit message standards
- **api-design**: RESTful API design patterns and conventions
</available-skills>
```

Skills metadata (~30-50 tokens per skill) is injected before each LLM request, while full content is loaded on-demand only.

### Configuration

**Disable automatic visibility** (manual discovery only):
```yaml
tools:
  - module: tool-skills
    config:
      visibility:
        enabled: false
```

**Limit visible skills** (for large skill sets):
```yaml
tools:
  - module: tool-skills
    config:
      visibility:
        max_skills_visible: 20  # Show top 20 alphabetically
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
def sample_skills():
    """Sample skills for testing."""
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
async def test_injects_skills_list(sample_skills):
    """Verify skills list is injected."""
    hook = SkillsVisibilityHook(sample_skills, {})
    result = await hook.on_provider_request("provider:request", {})
    
    assert result.action == "inject_context"
    assert "<available-skills>" in result.context_injection
    assert "python-testing" in result.context_injection
    assert "git-workflow" in result.context_injection

@pytest.mark.asyncio
async def test_respects_enabled_flag(sample_skills):
    """Verify hook can be disabled."""
    hook = SkillsVisibilityHook(sample_skills, {"enabled": False})
    result = await hook.on_provider_request("provider:request", {})
    
    assert result.action == "continue"

@pytest.mark.asyncio
async def test_handles_empty_skills():
    """Verify graceful handling of no skills."""
    hook = SkillsVisibilityHook({}, {})
    result = await hook.on_provider_request("provider:request", {})
    
    assert result.action == "continue"

@pytest.mark.asyncio
async def test_limits_max_visible():
    """Verify max_skills_visible limit."""
    many_skills = {
        f"skill-{i:03d}": SkillMetadata(
            name=f"skill-{i:03d}",
            description=f"Test skill {i}",
            path=Path(f"/skills/skill-{i:03d}/SKILL.md"),
            source="/skills",
        )
        for i in range(100)
    }
    
    hook = SkillsVisibilityHook(many_skills, {"max_skills_visible": 10})
    result = await hook.on_provider_request("provider:request", {})
    
    assert "skill-000" in result.context_injection  # Should show first 10
    assert "(90 more" in result.context_injection   # Should show truncation
    assert "skill-050" not in result.context_injection  # Should not show beyond limit
```

---

## Implementation Summary

### Changes Required (tool-skills repo only)

| File | Change | Lines |
|------|--------|-------|
| `amplifier_module_tool_skills/hooks.py` | CREATE | ~80 |
| `amplifier_module_tool_skills/__init__.py` | MODIFY | +15 |
| `behaviors/skills.yaml` | MODIFY | +10 |
| `bundle.md` | MODIFY | +25 |
| `tests/test_hooks.py` | CREATE | ~100 |

**Total**: ~230 lines of new code

### No Changes Needed

- ✅ amplifier-core (kernel)
- ✅ amplifier-foundation (bundle system)
- ✅ Any other repositories
- ✅ No new repos to create

---

## Token Cost Analysis

**Per skill overhead**: ~40 tokens (name + description)
- `- **python-testing**: Best practices for Python testing` ≈ 12-15 tokens

**Typical scenarios**:
- 10 skills = ~400 tokens per request
- 25 skills = ~1,000 tokens per request
- 50 skills = ~2,000 tokens per request (use limit)
- 100+ skills = Configure max_visible: 20

**Comparison to current**:
- Current: Agent must call load_skill(list=true) first = 1 tool call + response
- New: Skills visible automatically = +400 tokens but no round-trip
- Net: Better UX, roughly token-neutral

---

## Design Rationale

### Why Integrated Hook (Not Separate Repo)

1. **Tight coupling**: Skills visibility is inherent to skills capability
2. **Data reuse**: Hook uses tool's `skills` dict (no duplicate discovery)
3. **Single feature**: Users include skills, they get visibility automatically
4. **Maintenance**: One repo to version, test, release
5. **Precedent**: `hooks-todo-display` in foundation (embedded hook)

### Why provider:request (Not session:start)

1. **Dynamic skills**: Skills can change during session
2. **Always current**: Agent sees latest skill set every turn
3. **Ephemeral option**: Don't bloat history with repeated lists
4. **Precedent**: hooks-todo-reminder uses same timing

### Configuration Flexibility

Users can:
- Disable entirely: `visibility: {enabled: false}`
- Limit list size: `max_skills_visible: 20`
- Change injection role: `inject_role: "system"`
- Persist in history: `ephemeral: false`

---

## Implementation Steps

1. **Create hooks.py** with SkillsVisibilityHook class
2. **Update __init__.py** to mount hook after tool
3. **Update behaviors/skills.yaml** with visibility config
4. **Update bundle.md** to document visibility feature
5. **Create test_hooks.py** with comprehensive tests
6. **Test end-to-end** to verify skills appear in context
7. **Update README.md** if needed (mention visibility feature)

---

## Success Criteria

- ✅ Skills list appears in agent context before each LLM call
- ✅ Agent can discover skills without calling load_skill(list=true)
- ✅ No changes required to core/foundation/other repos
- ✅ Configurable (can be disabled or customized)
- ✅ Token cost reasonable (<1k tokens for 25 skills)
- ✅ Follows established patterns (hooks-todo-reminder)

---

## Open Questions

1. **PR Strategy**: Include in current bundle-integration PR or separate PR?
2. **Default enabled**: Should visibility be on by default? (Recommendation: yes)
3. **Format options**: Start with compact, add verbose later?

---

## Next Steps

**Ready to implement.** Waiting for decision on:
- Include in current PR or separate PR?
- Any concerns about the approach?
