---
bundle:
  name: test-local-skills
  version: 1.0.0
  description: Test bundle using local development version with hooks.py

includes:
  - bundle: git+https://github.com/microsoft/amplifier-foundation@main

tools:
  - module: tool-skills
    source: file:///Users/robotdad/Source/Work/skills/amplifier-module-tool-skills
    config:
      visibility:
        enabled: true
---

# Test Local Skills Bundle

This bundle loads the LOCAL development version of tool-skills (with hooks.py).

---

@foundation:context/shared/common-system-base.md
