---
name: to-specs
description: Turn the current conversation into one or more specs.
disable-model-invocation: true
---

This skill takes the current conversation context and codebase understanding and produces one or more specs (you may know this document as a PRD). Do NOT interview the user — just synthesize what you already know.

The specs should be saved to `docs/specs/<feature-folder>` where the `feature-folder` is an existing or a new folder that categorizes the specs.

## Process

1. Explore the repo to understand the current state of the codebase, if you haven't already. Use the project's domain glossary vocabulary throughout the specs, and respect any ADRs in the area you're touching.

2. Sketch out the seams at which you're going to test the feature. Existing seams should be preferred to new ones. Use the highest seam possible. If new seams are needed, propose them at the highest point you can. The fewer seams across the codebase, the better - the ideal number is one.

3. Write the specs.
