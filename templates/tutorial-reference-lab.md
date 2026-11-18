---
tags:
  - Lab
  - <LEVEL>
  - <TOPIC>
---

<!--
TEMPLATE: Tutorial Reference lab
See templates/README.md for how to use this file.

Replace every <PLACEHOLDER>. Delete HTML comments (like this one) and any
optional block you don't need before publishing.

Path variable used below (see README "Relative links"):
  <TUTORIAL_PATH>  relative path from this page to the tutorials folder,
                   for example ../../../containers

HOW A STEP IS WRITTEN
Every step starts with a GOAL and ends with a CHECK. In between, choose one
pattern depending on what the tutorial covers:

  A. Covered           -> Reference link only. The learner builds the command.
  B. Partly or not     -> Reference link for what is covered, then the command
     covered              with an explanation (the "assisted" pattern).
  C. Manual step       -> No Reference and no command.
-->

# <TOPIC> Lab: Tutorial Reference

<span class="lab-badge"><MINUTES> min</span> <span class="lab-badge"><LEVEL></span>

<ONE_SENTENCE_LAB_SUMMARY: what the learner will build, run or deploy.>

This lab uses a **tutorial reference** format. Each step gives you a goal and points to the section of the [<TUTORIAL_TITLE>](<TUTORIAL_PATH>/<TUTORIAL_FILE>.md) that shows the command you need. **You build the command yourself** by adapting the tutorial's example to this lab. When the tutorials don't cover something, the step gives you the command and explains what it does.

!!! tip "Keep the tutorials open"
    Open the [<TUTORIAL_TITLE>](<TUTORIAL_PATH>/<TUTORIAL_FILE>.md) in a second tab before you start. <ONE_SENTENCE_ON_HOW_TUTORIAL_EXAMPLES_DIFFER_FROM_THIS_LAB, for example: "This lab uses `greeting` where the tutorial uses `my-web-app`.">

## Prerequisites

- <PREREQUISITE_1>
- <PREREQUISITE_2>

## Part 1: <PART_TITLE>

<!--
PATTERN A: the tutorial covers the command.
Link the section and say what to change from the tutorial's example.
Do not include the command.
-->

1. <GOAL: what the learner must achieve, written as an instruction. Do not include the command.>
    - **Reference:** [<TUTORIAL_TITLE>, step <N>](<TUTORIAL_PATH>/<TUTORIAL_FILE>.md#<anchor>). <OPTIONAL: what to change from the tutorial's example, for example "Use `-e` to set a variable".>
    - **Check:** <OBSERVABLE_RESULT the learner can verify, for example output text or state.>

<!--
PATTERN B: the tutorial does NOT cover the command, or covers only part of it.
This is the "assisted" pattern: give the command and explain it, so the
learner understands what they are running. Use it for the missing part only.
If the tutorial covers part of the step, link that part first.

Include, in this order:
  1. Reference line: what the tutorial covers and what it doesn't.
  2. The command in a code block.
  3. An explanation. Use a "Part | What it does" table when the command has
     two or more parts to explain. Use one or two sentences when it has one.
  4. Expected output, if the learner can't tell from the Check whether the
     command worked. Omit it when the Check already says what to look for.
  5. Check.
-->

2. <GOAL>
    - **Reference:** [<TUTORIAL_TITLE>, step <N>](<TUTORIAL_PATH>/<TUTORIAL_FILE>.md#<anchor>) shows <WHAT_IS_COVERED>. It doesn't cover <WHAT_IS_MISSING>, so use this command:

    ```bash
    <COMMAND>
    ```

    | Part | What it does |
    | --- | --- |
    | `<COMMAND_OR_SUBCOMMAND>` | <WHAT_IT_DOES> |
    | `<FLAG_OR_ARGUMENT>` | <WHAT_IT_DOES> |

    ```text title="Expected output"
    <EXAMPLE_OUTPUT>
    ```

    - **Check:** <OBSERVABLE_RESULT>

<!--
PATTERN B, shorter form: one flag missing from a covered command.
Use this when the learner only needs one extra piece.
-->

3. <GOAL>
    - **Reference:** [<TUTORIAL_TITLE>, step <N>](<TUTORIAL_PATH>/<TUTORIAL_FILE>.md#<anchor>) shows <WHAT_IS_COVERED>. It doesn't cover <WHAT_IS_MISSING>, so add this flag: `<FLAG>`
    - **Check:** <OBSERVABLE_RESULT>

<!--
PATTERN C: manual step (for example a click in a web console).
Leave out Reference. Keep Check if there is something to verify.
-->

4. <GOAL>
    - **Check:** <OBSERVABLE_RESULT>

## Part 2: <PART_TITLE>

<!--
FILES THE LEARNER MUST CREATE (assisted pattern).
Give the file, then explain it line by line or instruction by instruction.
Include a source file from the repo with the snippet syntax, so the page and
the file never drift apart:
-->

### Create <FILE_OR_RESOURCE_NAME>

<ONE_SENTENCE_ON_WHAT_THE_FILE_IS_FOR.>

```<LANG> title="<FILENAME>"
--8<-- "<PATH_FROM_REPO_ROOT_TO_FILE>"
```

<!--
Explain each instruction, key, or flag that the tutorials don't already cover.
Delete bullets for items the tutorial explains and link the tutorial instead.
-->

- `<INSTRUCTION_OR_KEY>` <WHAT_IT_DOES>
- `<INSTRUCTION_OR_KEY>` <WHAT_IT_DOES>

<!-- Optional: one paragraph on the concept the file demonstrates. -->

5. <GOAL>
    - **Reference:** [<TUTORIAL_TITLE>, step <N>](<TUTORIAL_PATH>/<TUTORIAL_FILE>.md#<anchor>)
    - **Check:** <OBSERVABLE_RESULT>

<!-- Repeat Parts and steps. Keep step numbers continuous across Parts. -->

<!--
OPTIONAL: tip and warning blocks. Put a tip before the steps it affects, and a
warning where a failure is likely. Both blocks can contain commands that the
tutorials don't cover, for example:

!!! tip "<TITLE>"
    <TEXT>

!!! warning "<TITLE>"
    <TEXT>

    ```bash
    <COMMAND>
    ```
-->

## Cleanup

<GOAL: what to remove or keep.>

- **Reference:** [<TUTORIAL_TITLE>, step <N>](<TUTORIAL_PATH>/<TUTORIAL_FILE>.md#<anchor>)

<WHAT_TO_KEEP_FOR_THE_NEXT_LAB, with a link, or delete this line.>

## What you learned

- [x] <LEARNING_OUTCOME_1>
- [x] <LEARNING_OUTCOME_2>
- [x] <LEARNING_OUTCOME_3>
