---
name: build-graph
description: Turn a goal, plan, prompt, procedure, or existing workflow into a portable dependency graph. Use when the user asks to build, draw, parallelize, optimize, or find the graph in a workflow; identify nodes, edges, diamonds, gates, or fake dependencies; or prepare work for graph execution by any agent or tool system.
---

# Build the Graph

Convert work into a vendor-neutral `workgraph/v1` manifest. Design the graph; execute it only when the user also asks to run it.

## Build

1. State the goal and the final observable deliverable.
2. Decompose the work into nodes with one job and one inspectable output each. Split a node when its parts can fail, finish, or be retried independently.
3. Add an edge `A -> B` only when B consumes A's output, A changes whether B should run, or B would be unsafe before A completes. Timing, habit, or prompt order alone is a fake edge.
4. Remove fake edges. Nodes with satisfied dependencies form a ready set and may run together.
5. Add checker nodes at risky fan-in boundaries. Make a checker validate upstream outputs and emit a gate decision; keep synthesis in a separate node.
6. Prefer a static directed acyclic graph. Use `mode: dynamic` only when discoveries must create work that cannot be represented with known branches. Bound dynamic growth with `max_new_nodes` and `max_depth`.
7. Add completion criteria, failure behavior, side-effect level, and required capabilities to every node.
8. Validate the graph, then present it in the output contract below.

## Node design

Use short stable IDs. Write tasks as actions and `done_when` as checkable facts.

* `kind`: `work`, `check`, `decision`, or `join`.
* `depends_on`: node IDs that must complete successfully first.
* `inputs`: user inputs or named outputs consumed by the node.
* `output`: one named artifact, value, decision, or change.
* `done_when`: exhaustive, observable completion criteria.
* `capabilities`: needed abilities such as filesystem, web, browser, code execution, or domain judgment. Name capabilities, not products.
* `side_effects`: `none`, `reversible`, or `irreversible`.
* `failure`: `retry`, `block_descendants`, `degrade`, or `stop_graph`, with a limit or condition.
* `when`: optional simple condition based on a decision node's named outcome.

Keep independent writes to the same resource out of the same parallel wave unless the graph includes isolation and a merge node.

## Checker design

Check only properties that can change the downstream decision. Common checks include completeness, relevance, internal consistency, provenance, confidence, schema conformance, and safe-to-merge status.

Emit structured results:

```yaml
status: pass | retry | drop | block
accepted: [node_id]
rejected:
  - node: node_id
    reason: specific failure
```

Make every downstream node depend on the checker when the checker is a gate. Include the accepted upstream outputs in that downstream node's `inputs`; an edge to the checker alone does not carry the original data.

## Portable manifest

Use this shape and omit unused optional fields:

```yaml
version: workgraph/v1
name: short-name
goal: Observable end state
mode: static

limits:
  max_parallel: auto
  max_new_nodes: 0
  max_depth: 0

nodes:
  - id: source_a
    kind: work
    task: Produce one defined result
    depends_on: []
    inputs: [user.request]
    output: source_a.result
    done_when:
      - Result contains the required fields
    capabilities: [domain_judgment]
    side_effects: none
    failure: retry once, then block_descendants

  - id: gate
    kind: check
    task: Validate source_a.result for downstream use
    depends_on: [source_a]
    inputs: [source_a.result]
    output: gate.decision
    done_when:
      - Every input is marked pass, retry, drop, or block with a reason
    capabilities: [domain_judgment]
    side_effects: none
    failure: stop_graph

  - id: deliver
    kind: join
    task: Produce the final deliverable from accepted inputs
    depends_on: [gate]
    inputs: [source_a.result, gate.decision]
    output: final.result
    done_when:
      - The result satisfies the graph goal
    capabilities: [domain_judgment]
    side_effects: none
    failure: retry once, then stop_graph
```

Treat `depends_on` as scheduling constraints and `inputs` as data flow. Keep both explicit; they often overlap but are not interchangeable.

## Validate

Confirm all of the following before handing off:

* Every node ID and output name is unique.
* Every dependency and input reference resolves.
* The static portion is acyclic.
* Every edge passes the real-dependency test.
* Every node can be retried or diagnosed independently.
* Parallel nodes do not contend for the same mutable resource.
* Each convergence point declares how inputs are checked and combined.
* Every branch either reaches a terminal output or is explicitly droppable.
* Terminal outputs collectively satisfy the goal.
* Dynamic expansion, side effects, and retries have finite bounds.
* The ASCII graph contains the same nodes and dependencies as the manifest.

Revise until every item passes. Ask a question only when a missing choice would materially change the graph; otherwise state the assumption.

## Output contract

Return, in order:

1. **Graph summary** — goal, chosen mode, parallel waves, and critical gates.
2. **ASCII graph** — always include a fenced `text` block. Use the manifest's node IDs, draw top-to-bottom, and make fan-out, parallel branches, fan-in, checkers, and decision labels visible. Use plain ASCII characters such as `[ ]`, `-`, `|`, `/`, `\`, `+`, and `>`. Include every dependency. For a dense graph, split it into labeled subgraphs and add an exact edge list. Use Mermaid only when the user separately asks for it.
3. **Manifest** — one complete `workgraph/v1` YAML block, ready for `execute-graph`.
4. **Assumptions and risks** — only material ones.

For a small graph, return the contract inline. For a reusable or project-level graph, save it as `workgraph.yaml` in the relevant project when file creation is authorized.
