# Manual Testing Steps for Skills Module

**TESTED AND WORKING** - These exact commands have been verified on 2026-01-04.

---

## Prerequisites: Clear Cache First

**CRITICAL**: If you previously loaded tool-skills from git, you MUST clear the cache first. The cached version may not have hooks.py and will prevent the local development version from loading.

```bash
# Clear any cached tool-skills versions
find ~/.amplifier/cache -type d -name "*tool-skills*" -exec sh -c 'find "$1" -depth -type f -delete && find "$1" -depth -type d -delete' _ {} \;
```

**Verify cache is cleared**:
```bash
find ~/.amplifier/cache -name "*tool-skills*" || echo "✓ Cache cleared"
```

---

## Quick Test (4 commands)

```bash
cd amplifier-module-tool-skills

# 1. Copy test fixtures
mkdir -p .amplifier && cp -r tests/fixtures/skills .amplifier/

# 2. Register the bundle (NOTE: requires file://$PWD prefix)
amplifier bundle add file://$PWD/ai_working/test-local-bundle.md

# 3. Test visibility - agent should see skills WITHOUT using load_skill tool
amplifier run --bundle test-local-skills "What skills do you see available? Don't use any tools, just list them."
```

**Expected**: Agent lists 3 skills WITHOUT calling load_skill tool:
- amplifier-philosophy
- module-development  
- python-standards

**This proves the visibility hook is working.**

---

## Cleanup

```bash
# 1. Remove test skills directory
cd amplifier-module-tool-skills
find .amplifier -type f -delete && find .amplifier -type d -depth -delete

# 2. Clean both registries
cat ~/.amplifier/registry.json | jq 'del(.bundles["test-local-skills"])' > ~/.amplifier/registry.json.tmp && mv ~/.amplifier/registry.json.tmp ~/.amplifier/registry.json

cat ~/.amplifier/bundle-registry.yaml 2>/dev/null | sed '/test-local-skills:/,/added_at:/d' > ~/.amplifier/bundle-registry.yaml.tmp && mv ~/.amplifier/bundle-registry.yaml.tmp ~/.amplifier/bundle-registry.yaml

# 3. Clear cache (prevents stale versions from interfering with next test)
find ~/.amplifier/cache -type d -name "*tool-skills*" -exec sh -c 'find "$1" -depth -type f -delete && find "$1" -depth -type d -delete' _ {} \;
```

**Verify**:
```bash
ls .amplifier 2>/dev/null || echo "✓ Clean"
cat ~/.amplifier/registry.json | jq -r '.bundles | keys[]' | grep test || echo "✓ Clean"
```

---

## Why file://$PWD is Required

**This FAILS**:
```bash
amplifier bundle add ai_working/test-local-bundle.md
# Error: No handler for URI: ai_working/test-local-bundle.md
```

**This WORKS**:
```bash
amplifier bundle add file://$PWD/ai_working/test-local-bundle.md
# ✓ Added bundle 'test-local-skills'
```

Amplifier requires the `file://` URI scheme for local bundle files.

---

## Troubleshooting: Module Caching Issue

**Symptom**: Bundle configured with `source: file:///path/to/local` but old version without hooks.py still loads.

**Root Cause**: Amplifier caches modules in `~/.amplifier/cache/`. If you previously loaded tool-skills from git, the cached version (without hooks.py) takes precedence over the local `file://` source.

**Solution**: Clear the cache before testing local development versions:
```bash
# Find cached versions
find ~/.amplifier/cache -name "*tool-skills*" -type d

# Clear them
find ~/.amplifier/cache -type d -name "*tool-skills*" -exec sh -c 'find "$1" -depth -type f -delete && find "$1" -depth -type d -delete' _ {} \;
```

**Verification**: After clearing cache and running test bundle:
- Skills should be visible without calling load_skill tool
- Agent should list 3 skills: amplifier-philosophy, module-development, python-standards
- load_skill tool should successfully load skill content

**Note**: Cache directories cannot be removed with `rm -rf` due to safety restrictions. Use the find command pattern shown above.
