---
title: Alchemiscale TaskRestartPolicy proposal
author: Ian Kenney
date: 2024-06-20
description: A proposal for bringing automatic Task restarts to Alchemiscale, decreasing the need for users to manually intervene when resubmitting Tasks.
---

## Introduction

Since tasks are run on a variety of compute resources, many modes of failure can be anticipated by a user and some of these modes simply require that the task is resubmitted.
Especially in the context of living networks and automated strategies, the need to manually restart tasks is needlessly cumbersome.
These concerns were originally raised by Jenke Scheen and [an issue][issue] was created.
Here I propose an architecture which automates restarting tasks.

## User requirements and expectations

A user with an appropriate `Scope` will have the ability to modify the `TaskRestartPolicy` attached to an `AlchemicalNetwork`s `TaskHub`.
This ability will be exposed with the `AlchemiscaleClient` methods:

* `add_task_restart_policy_patterns(an_sk: ScopedKey, patterns: List[str], num_allowed_errors: int) -> ScopedKey`
* `get_task_restart_policy_patterns(an_sk: ScopedKey) -> Dict[str, int]`
* `remove_task_restart_policy_patterns(an_sk: ScopedKey, patterns: List[str])`
* `clear_task_restart_policy(an_sk: ScopedKey)`

For instance the code block,

```python
add_task_restart_policy_patterns(scoped_key, ["string1", "string2", "string3"], 5)
add_task_restart_policy_patterns(scoped_key, ["string1", "string4", "string5"], 3)
```

would result in the following set of policies (as reported by `get_task_restart_policy_patterns`, likely in a different order):

```
{
"string1": 3,
"string2": 5,
"string3": 5,
"string4": 3,
"string5": 3,
}
```

Note that since `string1` appeared in both calls, the assigned allowed number of errors is set to the latest value.
Along the same line, restart policy patterns can be removed with:

```python
remove_patterns = ["string2", "string3"]
remove_task_restart_policy_patterns(scoped_key, remove_patterns)
```

## Internal database representation

`TaskRestartPolicy`s attach directly to a `TaskHub` with the `ENFORCES` relationship.
This node only contains the policy strings and their number of allowed failures.

`Task` histories are tracked with `TaskHistory`s which `RECORDS` the `Task`.
When a task fails, the tracebacks are pulled from the PDR units and recorded here.
Additional information may be added to these histories later, but is out of scope for this feature.

When a `ComputeService` sends back data from a failed `Task`, if a `TaskRestartPolicy` is connected to the appropriate `TaskHub`, then the `Task` status is switched to `PendingError`.
Alternatively, the status could be immediately set to waiting here if appropriate.


## Execution of restarts

Like the rest of the `Alchemiscale` architecture, the restarting of `Task`s will be handled by a dedicated `TaskRestartService`.
The only purpose of this service is to periodically nudge the API server to the `alchemiscale.storage.statestore.Neo4jStore.tasks_auto_restart` method.
During the execution of this method, first Neo4J is queried for **all** `TaskRestartPolicy` data, shown [above](#internal-database-representation).
Second, all `Task`s in the `PendingError` state are queried along with their `TaskHistory`s.
Comparing this data with that in the `TaskRestartPolicy` informs the method whether or not to mark the `Task` as waiting.
If a `Task` in `PendingError` state has already been restarted `num_allowed_restarts` times, then the `Task` is marked as a full `Error` state.

[issue]: https://github.com/openforcefield/alchemiscale/issues/277
