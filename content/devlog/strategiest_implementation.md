---
title: Alchemiscale strategies proposal
author: Ian Kenney
date: 2024-08-26
description: A design for strategies in Alchemiscale
---

## Introduction

In its current state, execution of Tasks performing Transformations in alchemiscale requires user to manually create any number of Tasks for each transformation of interest and action those Tasks through a TaskHub (AlchemicalNetwork).
For smaller networks, this is manageable.
However, as compute pools increase and networks grow, the need for automation is apparent.
Here we propose the implementation of strategies, which can spawn and action Tasks for an AlchemicalNetwork.

## Feature description

An introduction of Strategies will include the implementation of the Strategist service in alchemiscale, which will be responsible for executing Strategies associated with a given AlchemicalNetwork in the Neo4j database.
The `Strategy`s themselves are implementations of `BaseStrategy`s that will be located in a dedicated library under the OpenFreeEnergy organization.
`Strategy`s will take as arguments an `AlchemicalNetwork` along with transformation free energy differences.
The output will be a `StrategyResult` object, a `GufeTokenizable` whose attributes reflect aspects of the `Strategy`'s proposal.

Using a `Strategy` directly (as the alchemiscale instance will do) will look something like:

```python
# given an AlchemicalNetwork (an) and transformation data (data)

strat: CustomStrategy = CustomStrategy(settings)

# propose method returns a StrategyResult object
results: StrategyResult = strat.propose(an, data)
weights: dict[GufeKey, float] = results.weights
```

Assuming a `Strategy` implementation is installed on both the client and the deployed alchemiscale server, typical usage client-side will look like:

```python
# with an AlchemicalNetwork ScopedKey (an_sk), AlchemiscaleClient (asc)

from customstrategylibrary import CustomStrategy

settings: CustomStrategySettings = CustomStrategy.default_settings()
strategy: CustomStrategy = CustomStrategy(settings)

strategy_scoped_key: ScopedKey = asc.set_network_strategy(an_sk, strategy)
```

Since `Strategy`s are immutable, any changes to a previously set `Strategy` need to be applied by first clearing the old strategy and re-set the new one.
The first functioning `Strategy` implemented will be take inspiration from the [Network Binding Free Energy (NetBFE)](https://pubs.acs.org/doi/10.1021/acs.jctc.1c00703) method, which aims to optimally allocate resources to the binding free energy calculations in the network.
The Strategist will use the `StrategyResult` to create and action new `Task`s for its associated `AlchemicalNetwork`, taking into consideration the current set of Tasks as well as their statuses.

### New client methods

We will introduce the following new client methods:

- `set_network_strategy(ScopedKey, type[StrategyBase])`
- `clear_network_strategy(ScopedKey)`
- `get_network_strategy(ScopedKey)`

### New statestore methods

The alchemiscale statestore module will have the same methods added to handle the database transactions:

- `set_taskhub_strategy(ScopedKey, type[StrategyBase])`
- `clear_taskhub_strategy(ScopedKey)`
- `get_taskhub_strategy(ScopedKey)`

## Design considerations

### Strategies

Strategies are opinionated in terms of their inputs and outputs as they need to be predictable and deterministic.
They require a `StrategySettings` object, a pydantic model, for initialization.
For transformation rankings (weights), they require the full `AlchemicalNetwork` along with data on computed transformations.
A `Strategy` proposal should always return the same output given the same input.

Ultimately, it is the job of the end user (or the Strategist service) to interpret the outputs of the resulting transformation rankings output by a `Strategy`'s propose method.

#### Representation in alchemiscale

Strategies are connected to `TaskHub`s with the `PROPOSES` relationship.
The Strategy node is a `GufeTokenizable` that contains settings that are specific to that Strategy.
Only one strategy can be associated with a TaskHub.
The nodes contain all information needed to import the strategy class, which are then stored in the `TOKENIZATION_CLASS_REGISTRY`.

### Strategist service

With the expected addition of [Task restart policies](../taskrestartpolicy), care needs to be taken so that no feedback loops appear as a result of the Strategist service which could waste time, compute, and money.
To combat this, we can declare some simple restrictions on how the Strategist updates the database.

The strategist service:

1. cannot switch the status of a Task from "error" to "waiting".
1. cannot create any Tasks on a Transformation with associated "errored" Tasks.
1. must count already existing tasks on the transformation against the count.
    - For example, if the strategist proposes that three Tasks should be run for Transformation-X, but two Tasks are already waiting, only one new Task will be created and actioned.
1. may action Tasks that are waiting

## Implementation plan

This feature requires we implement both the core Strategy structure and the Strategist service.
The service, being specific to alchemiscale will be developed in the [alchemiscale repository](https://github.com/openforcefield/alchemiscale) strategies, being general and of use outside of alchemiscale, will be developed outside of this repository.

Since the inputs and outputs of a Strategy are straightforward, these two efforts can be developed mostly in parallel with the Strategy library (namely the initial implementation of NetBFE) being a bottleneck for testing in alchemiscale.

### Strategy

Given their common `GufeTokenizable` base class, `Strategy`s will have a similar structure to the `gufe` implemented `Protocol` class.

```python
class StrategyBase(GufeTokenizable):

	def __init__(self, settings: type[StrategySettings]):
		self._settings = settings
	
	@classmethod
	def _defaults(cls):
		return {}
	
	def _to_dict(self) -> dict:
		return {'settings': self.settings}
	
	def _from_dict(cls, dct: dict):
		return cls(**dct)

	@property
	def settings(self) -> type[StrategySettings]:
		return self._settings
	
	@classmethod
	def default_settings(cls) -> type[StrategySettings]:
		return cls._default_settings()
	
	# abstract methods for strategy developers to implement
	@classmethod
	@abc.abstractmethod
	def _default_settings(cls) -> type[StrategySettings]:
		raise NotImplementedError
	
	@abc.abstractmethod
	def propose(
		self, 
		alchemical_network: AlchemicalNetwork, 
		transformation_data: dict[GufeKey, list[tuple[float, float]]]
	) -> StrategyResult:
		raise NotImplementedError
```

### StrategyResult

In addition to the `StrategyBase`, we will also implement a `StrategyResult`.
Minimally, this class is used to communicate the weights of a `Strategy` proposal, but *could* include other properties if implemented.

```python
class StrategyResult(GufeTokenizable):

	def __init__(self, **data):
		self._results = data
	
	@classmethod
	@abc.abstractclass
	def _defaults(cls):
		# could be as simple as {"weights": None}
		raise NotImplementedError
	
	def _to_dict(self):
		return self._results
	
	@classmethod
	def _from_dict(cls, dct: dict)
		return cls(**dict)

	@property
	def weights(self):
		return self._weights
```

### Alchemiscale Strategist

The Alchemiscale Strategist service will run periodically on all `AlchemicalNetwork`s that have an associated `Strategy`.

## Documentation

A new documentation site will be deployed for the strategies package.
This will include API documentation as well as a basic user/developer guide.
The [alchemiscale docs](https://docs.alchemiscale.org/en/latest/) will be updated to include basic usage of strategies.
