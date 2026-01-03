# Manual Testing Steps for Skills Module

Follow these steps to test bundle integration, skills visibility, and Anthropic spec compliance.

---

## Quick Test (Simple)

### Testing (3 commands)

```bash
cd /Users/robotdad/Source/Work/skills/amplifier-module-tool-skills

# Copy test fixtures to default location
mkdir -p .amplifier && cp -r tests/fixtures/skills .amplifier/

# Register and test
amplifier bundle add file://$PWD
amplifier run --bundle skills "What skills do you see available? Don't use any tools, just tell me what you see in your context."
```

**Expected**: Agent should see and list the 3 skills WITHOUT calling load_skill tool:
- amplifier-philosophy
- module-development  
- python-standards

This proves the visibility hook is injecting skills into context automatically.

### Cleanup (3 commands)

```bash
# Remove bundle
amplifier bundle remove skills

# Clean persisted registry
cat ~/.amplifier/registry.json | jq 'del(.bundles["skills"], .bundles["skills-behavior"])' > ~/.amplifier/registry.json.tmp && mv ~/.amplifier/registry.json.tmp ~/.amplifier/registry.json

# Remove test fixtures
rm -rf .amplifier
```

### Verify

```bash
amplifier bundle list | grep skills
```

**Expected**: No output

---

## Comprehensive Testing

### Part 1: Bundle Integration

#### 1. Navigate to directory

```bash
cd /Users/robotdad/Source/Work/skills/amplifier-module-tool-skills
```

#### 2. Copy test fixtures

```bash
mkdir -p .amplifier && cp -r tests/fixtures/skills .amplifier/
```

#### 3. Register bundle

```bash
amplifier bundle add file://$PWD
```

**Expected**: `✓ Added bundle 'skills'`

#### 4. Verify registration

```bash
amplifier bundle list | grep skills
```

**Expected**: Shows `skills` and `skills-behavior` bundles

---

### Part 2: Skills Visibility Testing

#### 5. Test automatic visibility (NEW FEATURE)

```bash
amplifier run --bundle skills "What skills do you see available? List them without using any tools."
```

**Expected**: Agent lists skills WITHOUT calling load_skill tool. This proves skills are visible in context automatically via the visibility hook.

**Example output**:
```
I can see these skills available:
- amplifier-philosophy: Amplifier design philosophy...
- module-development: Guide for creating new Amplifier modules...
- python-standards: Python coding standards...
```

#### 6. Test load_skill tool still works

```bash
amplifier run --bundle skills "Use the load_skill tool to list available skills"
```

**Expected**: Tool executes and returns same 3 skills.

#### 7. Test loading full skill content

```bash
amplifier run --bundle skills "Load the amplifier-philosophy skill and tell me the first key point"
```

**Expected**: Agent calls load_skill(skill_name="amplifier-philosophy") and reads the full content.

---

### Part 3: Spec Compliance Testing

#### 8. Test compatibility field (NEW FEATURE)

Create a test skill with compatibility:

```bash
mkdir -p .amplifier/skills/test-compat
cat > .amplifier/skills/test-compat/SKILL.md << 'EOF'
---
name: test-compat
description: Test compatibility field
compatibility: Requires git and docker
---
# Test Skill
EOF

amplifier run --bundle skills "Use load_skill with info='test-compat' to show metadata"
```

**Expected**: Output includes `"compatibility": "Requires git and docker"`

#### 9. Test allowed-tools parsing (FIXED)

Create skill with allowed-tools as string:

```bash
mkdir -p .amplifier/skills/test-tools
cat > .amplifier/skills/test-tools/SKILL.md << 'EOF'
---
name: test-tools
description: Test allowed-tools parsing
allowed-tools: bash read_file write_file
---
# Test Skill
EOF

amplifier run --bundle skills "Use load_skill with info='test-tools' to show metadata"
```

**Expected**: Output includes `"allowed_tools": ["bash", "read_file", "write_file"]` (parsed as list)

---

### Part 4: Behavior Inclusion Testing

#### 10. Test behavior inclusion in custom bundle

```bash
cat > ai_working/test-custom.md << 'EOF'
---
bundle:
  name: test-custom
  version: 1.0.0
includes:
  - bundle: git+https://github.com/microsoft/amplifier-foundation@main
  - bundle: file:///Users/robotdad/Source/Work/skills/amplifier-module-tool-skills#subdirectory=behaviors/skills.yaml

tools:
  - module: tool-skills
    config:
      skills_dirs:
        - /Users/robotdad/Source/Work/skills/amplifier-module-tool-skills/tests/fixtures/skills
---
# Test Custom Bundle
---
@foundation:context/shared/common-system-base.md
EOF

amplifier bundle add file://$PWD/ai_working/test-custom.md
amplifier run --bundle test-custom "What skills are available? Don't use tools."
```

**Expected**: Skills visible via behavior inclusion (same as main bundle)

---

## Part 5: Cleanup

### 11. Remove bundles from user registry

```bash
amplifier bundle remove skills
amplifier bundle remove test-custom
```

### 12. Clean foundation's persisted registry

```bash
cat ~/.amplifier/registry.json | jq 'del(.bundles["skills"], .bundles["skills-behavior"], .bundles["test-custom"])' > ~/.amplifier/registry.json.tmp && mv ~/.amplifier/registry.json.tmp ~/.amplifier/registry.json
```

### 13. Remove test files

```bash
rm -rf .amplifier
rm -f ai_working/test-custom.md
```

### 14. Verify cleanup

```bash
amplifier bundle list | grep -E "skills|test"
cat ~/.amplifier/registry.json | jq -r '.bundles | keys[]' | grep -E "skills|test"
```

**Expected**: Both commands return no output

---

## What's Being Tested

### Bundle Integration (Original PR)
- ✅ behaviors/skills.yaml enables composability
- ✅ bundle.md showcases thin bundle pattern
- ✅ README is bundle-first
- ✅ Examples converted to bundles

### Skills Visibility (NEW - This PR)
- ✅ Skills automatically visible in agent context (no tool call needed)
- ✅ Hook injects `<available_skills>` before each LLM request
- ✅ Progressive disclosure: metadata visible, content loaded on demand
- ✅ Configurable (enabled by default)

### Anthropic Spec Compliance (NEW - This PR)
- ✅ `compatibility` field support
- ✅ `allowed-tools` parsing (string → list)
- ✅ Both fields exposed in info mode
- ✅ Full spec compliance

---

## Test Fixtures

Using existing `tests/fixtures/skills/`:
- **amplifier-philosophy**: Amplifier design philosophy
- **module-development**: Guide for creating modules
- **python-standards**: Python coding standards

No need to create skills manually - fixtures are comprehensive.

---

## Troubleshooting

### Skills not visible in context

Check session logs to verify hook is firing:
```bash
# Find most recent session
SESSION_DIR=$(ls -t ~/.amplifier/projects/*/sessions/ | head -1)

# Check for skills:discovered event
grep "skills:discovered" $SESSION_DIR/events.jsonl

# Check for available_skills injection
grep "available_skills" $SESSION_DIR/events.jsonl
```

### Bundle not loading

```bash
# Check bundle status
amplifier bundle show skills

# Verify skills module is in tools list
amplifier bundle show skills | grep tool-skills
```
