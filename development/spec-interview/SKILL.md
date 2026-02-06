---
name: spec-interview
description: |
  Use this skill if a user's request is under-specified, and you need to clarify the project requirements.
---

# Overview

Turn project ideas into a fully-formed project specification by asking the user sets of questions until all ambiguity is removed.
This skill helps establish a complete context of a new project's design, and captures critical context before implementation is begun.

## Prerequisites

The user provides a high-level project design idea.
The user MAY also provide additional context about technical constraints, or scenarios the project must avoid.
Keep track of the current design by maintaining a file SPECIFICATION.md,
and generate that file if it doesn't exist.

## Workflow

### Understand Design Goals

- Start by loading the current design from SPECIFICATION.md
- Ensure the high-level project design is consistent with all section titles of all sections below
- For each section, read the entire section and think of a question with a non-obvious answer for that section to ask the user
- If the user says that the question is not relevant, ignore that question and any like it
- If the user provides an answer to that question, add it to the section in the correct location for it to read well
- If there are no more questions to ask, move on to the next section

### Exploring Alternatives

- If a user is unsure of the answer to a question, research 2-3 different approaches to the problem with their corresponding trade-offs
- Present each option conversationally and include relevant reasoning
- Explain your recommendation and ask if any clarification is needed
- Once the user selects an option, use that as your answer before moving on

### Summarizing the Design

- Once you have completely understood the design, summarize it and present that summary to the user
- Make sure to cover overall architecture and components, their data flow, any error testing, and how the correctness of the design can be verified
- Ask if any parts of the design need additional clarification before moving on

## Key Principles

- **Keep it simple** - Remove unnecessary features from the design, unless specifically requested
- **Explore alternatives** - Propose 2-3 different approaches before asking to settle on one
- **One question at a time** - Don't overwhelm the user with multiple questions at once
- **Prefer multiple choice questions** - It is easier to answer vs. open-ended questions
- **Verify before proceeding** - Make sure the user consents before moving on to a new topic
