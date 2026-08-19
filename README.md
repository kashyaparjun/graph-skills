# Graph Skills

Two portable agent skills for turning complex work into a dependency graph and running that graph to completion.

- **`build-graph`** designs a vendor-neutral `workgraph/v1` manifest from a goal, plan, prompt, procedure, or existing workflow.
- **`execute-graph`** validates and runs a `workgraph/v1` manifest as a dependency scheduler, with bounded retries, explicit gates, safe concurrency, and resumable state.

Together, they separate planning from execution:

```text
goal or workflow
      |
      v
[build-graph] --> workgraph/v1 manifest --> [execute-graph] --> verified result
```

## Why use a work graph?

Sequential checklists often hide work that can run in parallel and dependencies that do not actually exist. A work graph makes both control flow and data flow explicit:

- Nodes have one job and one inspectable output.
- Edges exist only for real dependencies.
- Independent nodes form a ready set and can run concurrently.
- Checker nodes gate risky fan-in boundaries.
- Completion criteria determine whether a node passed.
- Failure, retry, side-effect, and capability requirements are declared up front.

The format is deliberately vendor-neutral. Executors resolve abstract capabilities such as `filesystem`, `web`, `code_execution`, or `domain_judgment` to whatever tools are available in their environment.

## Installation

Clone the repository:

```bash
git clone https://github.com/kashyaparjun/graph-skills.git
cd graph-skills
```

Then copy or symlink each skill directory into the skills directory used by your agent.

For Codex:

```bash
mkdir -p ~/.codex/skills
ln -s "$(pwd)/build-graph" ~/.codex/skills/build-graph
ln -s "$(pwd)/execute-graph" ~/.codex/skills/execute-graph
```

For a project-local installation, place both directories in the skill location recognized by that project or agent runtime. Each directory is self-contained and uses the standard `SKILL.md` entrypoint.

Restart or reload the agent after installation if it does not discover new skills automatically.

## Usage

Build a graph from a goal:

```text
Use build-graph to turn this release procedure into a dependency graph.
```

Run a graph from a file:

```text
Use execute-graph to run workgraph.yaml to completion.
```

Build and execute in one request:

```text
Build a work graph for this migration, validate it, and execute it.
```

The build skill returns:

1. A graph summary.
2. An ASCII graph.
3. A complete `workgraph/v1` YAML manifest.
4. Material assumptions and risks.

The execution skill returns:

1. The terminal result or links to produced artifacts.
2. A concise run summary.
3. Any structural repairs or dynamic expansions.
4. Unresolved blockers and the smallest next action, when incomplete.

## Minimal manifest

```yaml
version: workgraph/v1
name: example
goal: A reviewed final deliverable exists
mode: static

limits:
  max_parallel: auto
  max_new_nodes: 0
  max_depth: 0

nodes:
  - id: draft
    kind: work
    task: Produce the first draft
    depends_on: []
    inputs: [user.request]
    output: draft.result
    done_when:
      - The draft covers every requested section
    capabilities: [domain_judgment]
    side_effects: none
    failure: retry once, then block_descendants

  - id: review
    kind: check
    task: Check the draft for completeness and consistency
    depends_on: [draft]
    inputs: [draft.result]
    output: review.decision
    done_when:
      - The draft is marked pass, retry, drop, or block with a reason
    capabilities: [domain_judgment]
    side_effects: none
    failure: stop_graph

  - id: deliver
    kind: join
    task: Produce the final deliverable from the accepted draft
    depends_on: [review]
    inputs: [draft.result, review.decision]
    output: final.result
    done_when:
      - The result satisfies the graph goal
    capabilities: [domain_judgment]
    side_effects: none
    failure: retry once, then stop_graph
```

`depends_on` controls scheduling. `inputs` declares data flow. They often overlap, but they are not interchangeable.

## Repository layout

```text
graph-skills/
|-- build-graph/
|   `-- SKILL.md
|-- execute-graph/
|   `-- SKILL.md
`-- README.md
```

## Design principles

- Prefer static directed acyclic graphs.
- Add an edge only when downstream work consumes an output, depends on a decision, or would be unsafe to start early.
- Keep validation in checker nodes and synthesis in separate join nodes.
- Make retries, dynamic expansion, cost, time, and side effects finite.
- Never treat graph approval as blanket authorization for destructive or unrelated actions.
- Finish only when every required terminal node has passed.

## Contributing

Keep the skills portable: capability names should describe abilities rather than products, and manifests should not depend on a particular agent vendor or orchestration platform.

When changing either skill, verify that examples remain valid, dependency and input references resolve, the static graph is acyclic, and the documented behavior matches the corresponding `SKILL.md`.
