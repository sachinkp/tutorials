# Task
You are updating architecture documentation for the <domain/subdomain>.

## Objective
<What must be corrected or improved in one sentence>

## Context
- System: <system name>
- Subdomain: <pre-approval / post-approval / etc.>
- Current pattern: <event-driven / synchronous>
- Source of truth: <Cosmos DB / Salesforce / etc.>

## Inputs
- Primary file: <path/to/file>
- Related files: <path1>, <path2>, <path3>
- Existing handler(s): <handler names>
- Existing event(s): <event names>

## Required Changes
1. <Change #1>
2. <Change #2>
3. <Change #3>

## Explicit Inclusions
- Include: <items that must appear>
- Keep: <things that must remain unchanged>

## Explicit Exclusions
- Exclude: <event/type/flow to remove>
- Do not introduce: <new patterns or assumptions to avoid>

## Flow Rules
1. <Publisher> publishes <event> to <topic/subscription>.
2. <Function/Handler> consumes and routes by <property, e.g., ChangeType>.
3. <API Controller> persists data into <target store>.
4. After persistence, publish <downstream sync event>.
5. <Sync handler> updates <external system> in <state, e.g., Prospect>.

## Consistency Requirements
- Use canonical event name: <event name>
- Use canonical handler name: <handler name>
- Keep numbering/labels consistent in diagrams
- Keep terminology consistent across all updated files

## Acceptance Criteria
- [ ] Diagram reflects required flow end-to-end
- [ ] Excluded events/types are removed
- [ ] Unified handler usage is clear
- [ ] Persistence happens before downstream sync event
- [ ] No contradictory terminology remains

## Output Format
- Provide:
  1. Updated content snippet(s)
  2. List of exact edits made
  3. Brief validation against acceptance criteria

## If Ambiguous
- Make the smallest safe assumption.
- State the assumption explicitly in one line.
