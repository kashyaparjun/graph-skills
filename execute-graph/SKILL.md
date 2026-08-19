---
name: execute-graph
description: Validate and run a portable dependency graph to completion using the tools, agents, processes, and permissions available in the current environment. Use when the user asks to execute, run, resume, or continue a work graph, DAG, node-and-edge plan, parallel workflow, or a workgraph/v1 manifest.
---

# Execute the Graph

Run a `workgraph/v1` manifest as a dependency scheduler, not as a sequential checklist. Preserve correctness, state, and authorization while exploiting safe concurrency.

## Accept

Accept a manifest inline, from a file, or from the immediately preceding response.

If the user provides only a workflow description, use `build-graph` first when available. Otherwise, construct the minimum equivalent manifest and show material assumptions before execution.

Treat:

* `depends_on` as scheduling constraints.
* `inputs` as explicit data flow.
* `done_when` as the node's acceptance test.
* `capabilities` as executor requirements, independent of any vendor.
* `side_effects` as a lower bound on safety handling, never as authorization.
* `failure` as the local recovery and propagation policy.

## Preflight

Before running any node:

1. Confirm the goal and terminal outputs.
2. Verify unique node IDs and output names.
3. Resolve every dependency and input reference.
4. Reject cycles in the static graph.
5. Confirm conditional nodes reference a reachable decision outcome.
6. Identify parallel nodes that could mutate the same resource; serialize them or require isolation plus an explicit merge.
7. Resolve each capability to an available executor. Prefer direct deterministic tools for deterministic work and reasoning agents for bounded judgment tasks.
8. Check that retry counts, dynamic growth, time, cost, and side effects are bounded.
9. Apply the current environment's approval and safety rules. Pause before a node that lacks required authority; approval for the graph is not blanket approval for unrelated or destructive actions.

Repair unambiguous structural defects and record the repair.

Stop for user input when a repair would change the goal, remove a required result, or expand authority.

## State

Maintain one run ledger with these node states:

```text
pending -> ready -> running -> passed | failed | blocked | dropped
```

Record for each attempted node:

* Attempt number and executor.
* Resolved input names.
* Start and finish state.
* Output location or inline value.
* Completion-criteria result.
* Failure or checker reason.

Use a persistent `workgraph-run.yaml` beside a reusable project graph when the run is long-lived, resumable, or produces files.

Keep the ledger in working memory for a small inline graph.

On resume, verify existing outputs before marking their nodes passed. Never trust stale status alone.

## Schedule

Repeat until no runnable work remains:

1. Mark a pending node ready when every required dependency has passed and its `when` condition, if any, is true.
2. Mark a false conditional branch dropped and record the decision.
3. Select up to `limits.max_parallel` ready nodes that do not contend for the same mutable resource.
4. Run independent nodes concurrently when the environment supports it. Use batched tool calls, isolated workers, or delegated agents as appropriate. Run sequentially when concurrency is unavailable or unsafe.
5. Give each executor only the graph goal, exact node task, resolved inputs, output contract, completion criteria, allowed side effects, and failure policy. Keep unrelated branches and expected conclusions out of the brief.
6. Inspect the returned output against every `done_when` criterion. Mark passed only when all criteria hold.
7. Store the named output and update the ledger before scheduling descendants.

Do useful coordinator work while delegated or background nodes run. Recompute the ready set after every state change.

## Gate

For a `kind: check` node, require a structured decision:

```yaml
status: pass | retry | drop | block
accepted: [node_id]
rejected:
  - node: node_id
    reason: specific failure
```

Apply it as follows:

* `pass`: release descendants with the accepted outputs.
* `retry`: rerun only the rejected producer nodes within their retry limits, then rerun the checker.
* `drop`: exclude rejected outputs only when downstream nodes permit degraded input; otherwise block them.
* `block`: mark affected descendants blocked and stop the impacted branch.

Keep validation and synthesis separate. A checker judges whether material may proceed; a join node combines accepted material.

## Fail

Apply the node's declared policy:

* `retry`: retry only after diagnosing the failed acceptance criterion; preserve the attempt history.
* `block_descendants`: continue unrelated branches and mark dependent nodes blocked.
* `degrade`: continue only when the downstream node's inputs and completion criteria permit the missing output.
* `stop_graph`: stop scheduling new work, preserve completed outputs, and report the blocker.

Never silently substitute a failed output, invent a missing artifact, or report a blocked terminal node as complete.

If no node is running or ready while required terminals remain pending, report a deadlock with the exact unsatisfied conditions.

## Expand a dynamic graph

Allow a completed node to propose new nodes only when `mode: dynamic`.

1. Require the proposal to state why the existing graph cannot finish the goal without the expansion.
2. Validate new IDs, outputs, dependencies, capabilities, side effects, and completion criteria.
3. Reject expansions that introduce a cycle, exceed `max_new_nodes` or `max_depth`, duplicate completed work, or require new authority.
4. Append accepted nodes and edges to the ledger before running them.

Keep completed history immutable. Record every accepted or rejected expansion and its reason.

## Finish

Finish only when every required terminal node has passed, or when progress is impossible.

Return:

1. **Result** — the terminal deliverable or direct links to its artifacts.
2. **Run summary** — passed, failed, blocked, dropped, and retried nodes.
3. **Changes to the graph** — repairs or dynamic expansions.
4. **Unresolved items** — blockers and the smallest next action, only when incomplete.

Lead with the result. Keep the ledger concise unless the user asks for a full trace.
