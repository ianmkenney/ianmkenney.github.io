---
title: Alchemiscale strategies proposal
author: Ian Kenney
date: 2024-08-26
description: A design for strategies in Alchemiscale
draft: true
---

## Introduction

In its current state, execution of Tasks performing Transformations in alchemiscale requires user to manually create any number of Tasks for each transformation of interest and action those Tasks through a TaskHub (AlchemicalNetwork).
For smaller networks, this is manageable.
However, as compute pools increase and networks grow, the need for automation is apparent.
Here we propose the implementation of strategies, which can spawn and action Tasks for an AlchemicalNetwork.

## Feature description

An introduction of Strategies will include the implementation of the Strategist service in alchemiscale, which will be responsible for executing Strategies associated with a given AlchemicalNetwork in the Neo4j database.
Strategies will take as arguments an AlchemicalNetwork along with current transformation free energy differences.
The output will be a StrategyResult object, a GufeTokenizable whose attributes reflect aspects of the Strategy's proposal.
The core Strategy structure will be implemented as a base class along with a derived class for the [Network Binding Free Energy (NetBFE)](https://pubs.acs.org/doi/10.1021/acs.jctc.1c00703) method, which aims to optimally allocate resources to the binding free energy calculations in the network.
The Strategist will use the output of a Strategy to create and action new Tasks for its associated AlchemicalNetwork, taking into consideration the current set of Tasks as well as their statuses.

```python
# an: AlchemicalNetwork
# data: dict
#   will contain relevant information related to the specific strategy
# settings: MyStrategySettings
#   setting specific to MyStrategy

strat: MyStrategy = MyStrategy(settings)

# propose method returns a StrategyResult object
results: StrategyResult = strat.propose(an, **data)
weights: dict[GufeKey, float] = results.weights

## user

from alchemistrategies import NetBFE
```

## Design considerations

### Strategies

Strategies are opinionated in terms of their inputs and outputs as they need to be predictable and deterministic.

#### Representation in alchemiscale

Strategies are connected to `TaskHub`s with the `PROPOSES` relationship.
The Strategy node is a `GufeTokenizable` with a string indicating the identity of the strategy, which will be specified in a strategy registry, along with settings that are specific to that Strategy.
When adding Strategies to `TaskHub`s this registry is checked and will return an error if the specified strategy could not be found.
An example of a registry could be:

```python
registry = {
	"NetBFE": ("alchemistrategies", "NetBFE"),
	"IansStrategy": ("ianstools.strategies", "IansStrategy"),
	...
}
```

where the classes can be dynamically loaded.

```python
from importlib import import_module

target_strategy = "NetBFE"

try:
	strategy_class = getattr(*registry[strategy])
except ImportError as e:
	...
```

Only one strategy can be associated with a TaskHub. 

### Strategist service

With the expected addition of [Task restart policies](../taskrestartpolicy), care needs to be taken so that no feedback loops appear as a result of the Strategist service which could waste time, compute, and money.
To combat this, we can declare some simple restrictions on how the Strategist updates the database.

The strategist service:

1. cannot switch the status of a Task from "error" to "waiting".
1. cannot create any Tasks on a Transformation with non-complete or non-waiting Tasks.
1. must count already existing tasks on the transformation against the count.
    - For example, if the Strategy states that three Tasks should be run for Transformation-X, but two Tasks are already waiting, only one new Task will be created and actioned.
1. may action Tasks that are waiting, but not actioned.

## Implementation plan

This feature requires we implement both the core Strategy structure and the Strategist service.
The service, being specific to alchemiscale will be developed in the [alchemiscale repository](https://github.com/openforcefield/alchemiscale) strategies, being general and of use outside of alchemiscale, will be developed outside of this repository.

Since the inputs and outputs of a Strategy are straightforward, these two efforts can be developed mostly in parallel with the Strategy library (namely the initial implementation of NetBFE) being a bottleneck for testing in alchemiscale.

## Testing

### Strategies

Tests will be provided for the abstract method and basic derived classes.
We will test proposed predictions on the tyk2 alchemical network, similar to the one found within the alchemiscale testing module.

Varying connectivities need to be considered while testing.

### Alchemiscale strategist

## Documentation

A new documentation site will be deployed for the strategies package.
This will include API documentations as well as a basic user guide.
