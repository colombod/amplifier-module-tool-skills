# Manual Testing Steps for Bundle Integration

Follow these steps in order to test and clean up the bundle integration.

---

## Part 1: Testing

### Step 1: Navigate to the skills module directory

```bash
cd /Users/robotdad/Source/Work/skills/amplifier-module-tool-skills
```

### Step 2: Register the local bundle

```bash
amplifier bundle add file://$PWD
```

**Expected**: `✓ Added bundle 'skills'`

### Step 3: Verify bundle is registered

```bash
amplifier bundle list | grep skills
```

**Expected**: Should show `skills` and `skills-behavior` bundles

### Step 4: Test the bundle end-to-end

The repo already has test skills in `tests/fixtures/skills/`. Configure bundle to use them:

```bash
cat > ai_working/test-skills-bundle.md << 'EOF'
---
bundle:
  name: test-skills-local
  version: 1.0.0
  description: Test bundle using fixture skills

includes:
  - bundle: git+https://github.com/microsoft/amplifier-foundation@main

tools:
  - module: tool-skills
    source: file:///Users/robotdad/Source/Work/skills/amplifier-module-tool-skills
    config:
      skills_dirs:
        - /Users/robotdad/Source/Work/skills/amplifier-module-tool-skills/tests/fixtures/skills
---

# Test Skills Bundle

---

@foundation:context/shared/common-system-base.md
EOF
```

### Step 5: Register and test with fixtures

```bash
amplifier bundle add file://$PWD/ai_working/test-skills-bundle.md
amplifier run --bundle test-skills-local "Use the load_skill tool to list available skills"
```

**Expected**: Should list the 3 test skills (amplifier-philosophy, module-development, python-standards)

### Step 6: (Optional) Test behavior inclusion

```bash
cat > ai_working/test-behavior.md << 'EOF'
---
bundle:
  name: test-behavior
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
# Test Behavior
---
@foundation:context/shared/common-system-base.md
EOF

amplifier bundle add file://$PWD/ai_working/test-behavior.md
amplifier run --bundle test-behavior "Use load_skill to list available skills"
```

**Expected**: Should work identically via behavior inclusion

---

## Part 2: Cleanup

### Step 7: Remove bundles from user registry

```bash
amplifier bundle remove skills
amplifier bundle remove test-skills-local
amplifier bundle remove test-behavior
```

### Step 8: Clean foundation's persisted registry

```bash
cat ~/.amplifier/registry.json | jq 'del(.bundles["skills"], .bundles["skills-behavior"], .bundles["test-skills-local"], .bundles["test-behavior"])' > ~/.amplifier/registry.json.tmp && mv ~/.amplifier/registry.json.tmp ~/.amplifier/registry.json
```

### Step 9: Remove test bundle files

```bash
rm -f ai_working/test-skills-bundle.md
rm -f ai_working/test-behavior.md
```

### Step 10: Verify all bundles are gone

```bash
amplifier bundle list | grep -E "skills|test"
```

**Expected**: No output

### Step 11: Verify registry is clean

```bash
cat ~/.amplifier/registry.json | jq -r '.bundles | keys[]' | grep -E "skills|test"
```

**Expected**: No output

### Step 12: Check git status

```bash
git status --short
```

**Expected**: Only documentation files in `ai_working/` that are already committed

---

## Summary

**Test fixtures used**: `tests/fixtures/skills/` (already in repo)
- amplifier-philosophy
- module-development  
- python-standards

**No files created in module itself** - all test bundles are in `ai_working/` and cleaned up

**Cleanup locations**:
- `~/.amplifier/bundle-registry.yaml` (user registry)
- `~/.amplifier/registry.json` (foundation's persisted registry)
- `ai_working/test-*.md` (temporary test bundles)
