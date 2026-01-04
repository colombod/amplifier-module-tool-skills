# Manual Testing Steps for Skills Module

**TESTED AND WORKING** - These steps have been verified end-to-end on 2026-01-04.

---

## Quick Test (4 commands)

```bash
cd amplifier-module-tool-skills

# 1. Copy test fixtures
mkdir -p .amplifier && cp -r tests/fixtures/skills .amplifier/

# 2. Register the bundle (auto-registers on first use)
amplifier bundle add ai_working/test-local-bundle.md

# 3. Test visibility - agent should see skills WITHOUT using load_skill tool
amplifier run --bundle test-local-skills "What skills do you see available? Don't use any tools, just list them."

# 4. Test load_skill tool
amplifier run --bundle test-local-skills "Use the load_skill tool to load the amplifier-philosophy skill"
```

**Expected Results**:

**Step 3** - Agent lists 3 skills WITHOUT calling load_skill tool:
- amplifier-philosophy
- module-development  
- python-standards

**Step 4** - Agent successfully loads skill content using the tool

**This proves**: Visibility hook works (skills in context) AND load_skill tool works

---

## Cleanup (3 commands)

```bash
# 1. Remove test skills directory
find .amplifier -type f -delete && find .amplifier -type d -depth -delete

# 2. Remove test skill from user directory (if exists)
find ~/.amplifier/skills/test-visibility-check -type f -delete 2>/dev/null || true
find ~/.amplifier/skills/test-visibility-check -type d -depth -delete 2>/dev/null || true

# 3. Clean registry
cat ~/.amplifier/registry.json | jq 'del(.bundles["test-local-skills"])' > ~/.amplifier/registry.json.tmp && mv ~/.amplifier/registry.json.tmp ~/.amplifier/registry.json
```

**Verify Cleanup**:
```bash
ls .amplifier 2>/dev/null || echo "✓ .amplifier directory removed"
ls ~/.amplifier/skills/ 2>/dev/null || echo "✓ No user skills directory"
cat ~/.amplifier/registry.json | jq -r '.bundles | keys[]' | grep test || echo "✓ No test bundles in registry"
```

**Expected**: All test artifacts removed

---

## What Gets Tested

- ✅ Skills visibility hook: Agent sees skills in context automatically
- ✅ Progressive disclosure: Metadata visible (Level 1), full content loaded on demand (Level 2)
- ✅ Local development version: Loads hooks.py from your working directory
- ✅ load_skill tool: Successfully loads and displays skill content
- ✅ Bundle registration: Local file:// source works correctly

---

## How Bundle Loading Actually Works

**IMPORTANT**: You cannot use bundle file paths directly. The workflow is:

1. **Register bundle** (happens automatically on first use):
   ```bash
   amplifier bundle add ai_working/test-local-bundle.md
   ```
   This registers the bundle by its `bundle.name` field (in this case: "test-local-skills")

2. **Use bundle by name**:
   ```bash
   amplifier run --bundle test-local-skills "..."
   ```

**What DOESN'T work**:
- ❌ `amplifier run --bundle ai_working/test-local-bundle.md` (relative path)
- ❌ `amplifier run --bundle /full/path/to/test-local-bundle.md` (absolute path)

**What WORKS**:
- ✅ `amplifier bundle add <path>` + `amplifier run --bundle <name>`
- ✅ First run auto-registers, subsequent runs use the registered name

---

## Bundle File Details

The test bundle (`ai_working/test-local-bundle.md`) configures:

```yaml
bundle:
  name: test-local-skills  # This becomes the bundle name to use
  
tools:
  - module: tool-skills
    source: file:///Users/robotdad/Source/Work/skills/amplifier-module-tool-skills
    # Uses absolute file:// URL to load LOCAL development version
```

**Key points**:
- `source: file://` loads your local code (including hooks.py changes)
- `bundle.name` determines what name to use with `--bundle` flag
- Bundle auto-registers on first use, stays registered until removed
