# Templates

Reusable page templates for the bootcamp. This folder is outside `docs/`, so MkDocs does not build or publish it.

| Template | Use it to |
| -------- | --------- |
| [`tutorial-reference-lab.md`](tutorial-reference-lab.md) | Write or convert a lab in the **tutorial reference** format |

## The tutorial reference format

A tutorial reference lab gives the learner a **goal** for each step plus a **link to the tutorial section** that shows the command. The learner builds their own command from the tutorial's example.

When no tutorial section covers a command, the step falls back to the **assisted** pattern: it gives the command, explains each part, and shows the expected output. That way the learner is never stuck on something the tutorials don't teach.

Every step has the same shape:

```text
Goal  ->  Reference (link, what to change)  ->  [command + explanation, only if not covered]  ->  Check
```

### The three step patterns

| Pattern | When | The step contains |
| ------- | ---- | ----------------- |
| **A. Covered** | The tutorial shows the command | Goal, Reference link (with what to change), Check |
| **B. Not covered, or partly covered** | The tutorial doesn't show the command, or shows only part | Goal, Reference link for what is covered, the command, an explanation, expected output (if needed), Check |
| **C. Manual** | A click in a web console, or anything with no command | Goal, Check |

#### Pattern A: covered

```markdown
1. Show the container's logs.
    - **Reference:** [Container Tool Tutorials, step 4](../../../containers/container-tool-tutorials.md#4-inspect-and-debug-a-running-container)
    - **Check:** you see `greeting service listening on :8080`.
```

The learner reads the tutorial section and writes `podman logs greeting` themselves.

#### Pattern B: not covered, or partly covered (the assisted pattern)

Shorter form, for one missing flag:

```markdown
2. Run the `quay.io/podman/hello` container. The container should be deleted when it exits.
    - **Reference:** [Container Tool Tutorials, step 3](../../../containers/container-tool-tutorials.md#3-run-a-container) shows how to run a container. It doesn't cover deleting a container on exit, so add this flag: `--rm`
    - **Check:** you see `Hello Podman World`, and `podman ps -a` doesn't list the container.
```

Full form, for a command the tutorials don't teach:

````markdown
3. Print only the container's status and IP address.
    - **Reference:** [Container Tool Tutorials, step 4](../../../containers/container-tool-tutorials.md#4-inspect-and-debug-a-running-container) shows the full configuration. It doesn't cover printing selected fields, so use this command:

    ```bash
    podman inspect greeting --format '{{.State.Status}} {{.NetworkSettings.IPAddress}}'
    ```

    | Part | What it does |
    | --- | --- |
    | `podman inspect greeting` | Prints the container's configuration as JSON |
    | `--format '...'` | Takes a Go template. Each `{{ }}` selects one field from the JSON |

    ```text title="Expected output"
    running 10.88.0.2
    ```

    - **Check:** the output is a single line such as `running 10.88.0.2`.
````

Rules for Pattern B:

- Give only the **missing part** as a command. Link the covered part.
- Explain with a **"Part | What it does" table** when there are two or more parts. Use a sentence when there is one.
- Add **expected output** when the learner can't tell from the Check whether it worked. Skip it when the Check already says what to look for.

#### Pattern C: manual

```markdown
4. On quay.io, open the new `greeting` repository and, under **Settings**, make it **public**.
    - **Check:** the repository shows a **Public** badge.
```

### Files the learner must create

When the lab needs a file (for example a source file or a Containerfile), use the assisted pattern: show the file, then explain each instruction the tutorials don't cover.

- Include the file from the repo with `--8<-- "path/from/repo/root"` so the page and the file never drift apart.
- Explain the instructions in a bullet list, one line each. Link the tutorial instead of explaining anything it already teaches.
- Add one paragraph on the concept the file shows, if there is one (for example, why a multi-stage build makes a smaller image).

The template has a ready-made section for this in Part 2.

### How to convert a lab

1. **Pick the tutorials.** Find the tutorial pages that teach the commands this lab uses. The lab should come after them in the course. If no tutorial exists, write one first.
2. **Copy the template** into the lab's folder and replace every `<PLACEHOLDER>`.
3. **Write each step as a goal.** Strip the command out of the existing step. If the lab already has command-by-command steps, the command becomes the thing you look up in the tutorial.
4. **Choose a pattern** for each step using the table above.
5. **Link to the exact section.** Use the heading anchor, for example `tutorial.md#3-run-a-container`. MkDocs builds anchors from the heading text: lowercase, spaces become hyphens, punctuation removed. A numbered heading `## 3. Run a container` gives `#3-run-a-container`.
6. **Say what to change.** If the tutorial's example differs from the lab, say so in the Reference line (for example, "Run `id` instead of `/bin/sh`").
7. **Write a Check** for every step that produces a result.
8. **Move anything the old lab explained** (files, concepts, warnings) into the matching section of the template. Keep explanations of things the tutorials already teach out of the lab.

### Relative links

Links to other pages are relative to the page that contains them. The number of `../` depends on where the lab lives. Count the folders between the lab page and `docs/`:

| Lab page location | Path to `docs/containers/` |
| ----------------- | -------------------------- |
| `docs/labs/containers/lab1/` | `../../../containers` |
| `docs/labs/<topic>/<lab>/` | `../../../<tutorial folder>` |
| `docs/labs/<topic>/<lab>/<subfolder>/` | `../../../../<tutorial folder>` |

Run `mkdocs build --strict` after converting a page. It reports broken links and missing anchors.

### Registering the page

- Add the page to `nav` in [`mkdocs.yml`](../mkdocs.yml).
- Link the lab from the course pages that point to it (for example the topic's lab index).

### Guidelines

- **Write goals, not commands.** If the step text contains the command and the tutorial already covers it, use Pattern A instead.
- **Prefer a link over a command.** Only give a command when the tutorials don't cover it.
- **Keep the tutorial and lab examples aligned** where you can. Use the same image and file names so the learner only needs to adapt, not translate.
- **Change tutorials and labs together.** When a tutorial section is renamed or reordered, update the Reference links that point at it. Search for the tutorial's filename in `docs/labs/`.
- **Podman and Docker.** The tutorials show both tools in tabs. Labs that give commands directly can use one tool and note that the other works the same.
- **Checklist before publishing:**
    - [ ] No `<PLACEHOLDER>` or HTML comments left
    - [ ] Every Reference link resolves
    - [ ] Every step has a Check, except manual steps with nothing to verify
    - [ ] Commands given directly are only for things the tutorials don't cover
    - [ ] Every directly given command is explained
    - [ ] The page is in `nav`
