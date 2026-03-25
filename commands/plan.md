---
description: Restate requirements, assess risks, and create step-by-step implementation plan. WAIT for user CONFIRM before touching any code.
---

# Plan Command

Invokes the **planner** agent to create a comprehensive implementation plan before writing any code.

## What This Command Does

1. **Restate Requirements** -- Clarify what needs to be built
2. **Identify Risks** -- Surface potential issues and blockers
3. **Create Step Plan** -- Break down implementation into phases
4. **Wait for Confirmation** -- MUST receive user approval before proceeding

## When to Use

- Starting a new feature
- Making significant architectural changes
- Working on complex refactoring
- Multiple files/components will be affected

**CRITICAL**: The planner agent will NOT write any code until you explicitly confirm the plan.
