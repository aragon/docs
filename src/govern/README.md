---
sidebar: auto
---

# Govern

![Aragon Govern header](/govern.png)

Welcome to Govern's documentation.

Govern is software for creating and governing organizations such as DeFi projects, open source projects, gaming guilds, cooperatives, nonprofits, clubs, companies, and any other type of organization you can imagine. It's Aragon's implementation of ERC-3000, the standard for binding off-chain voting.

Along with off-chain voting solutions like [Vocdoni](https://docs.vocdoni.io/) it allows you to _govern_ all parts of your project, even if not completely decentralized, in an easy manner. Also allowing for flexibility and extensions when needed.

If you have any questions or you'd like to say hi, come join the community at [Discord](https://discord.com/invite/aragon)!

## Introduction

### Anatomy of a Govern DAO

A Govern DAO, in its most normal and basic form has two key pieces:

* An Action Queue
* An Executor

Govern's Action Queue is what most users will interact with directly—it holds the DAOs configuration parameters, and its where actors can schedule, execute, challenge, and veto actions. It's also flexible, meaning that actions can be introduced to tweak its own parameters on the fly.

Through the Executor, Govern DAOs can also hold funds and interact with any arbitrary protocol. This makes it equivalent to **aragonOS's Agent** and **Moloch's Minion.** This allows for complex user flows such as a community deciding through a Vocdoni vote to use pooled funds in the Executor to gain interest through lending protocols like Compound or AAVE and also fund a Balancer liquidity pool, all in one transaction.

Learn more about the [smart contract system here](#smart-contracts).

Govern will also have first class support with Vocdoni and Snapshot, in which a space or token will be able to be configured so that every proposal can attach an on-chain action to the corresponding options to vote for. With these tools, we have everything we need to allow a project, to fully and progressively decentralize and take decisions over its future in a community-oriented manner.

### Optimistic by design

Govern builds on the "Optimistic" concept, in which we assume actors are acting rationally and with good intentions, until proven otherwise. Actors with permissions in the DAO can submit actions, which can get executed after a defined time window. These can also be disputed with the chosen mechanism for resolving these challenge/response games, most likely a [subjective oracle](https://aragon.org/blog/snapshot), such as Aragon Court. These actions can also be vetoed by an actor with the corresponding permissions as long as they haven't been executed.

With the shift towards operating a DAO in an optimistic way, economic and social incentives are introduced for people to act rationally: any malicious actions that get submitted can get challenged, resulting in the malicious actor losing money and the challenger earning the actor's collateral. Unnecessary challenges are also disincentivized due to the need to lock collateral to do so and potential loss of reputation if a subjective oracle is used and fails in favor of the submitter.

To date, acceptable actions and behavioral boundaries within DAOs are rigidly defined in code, and strictly enforced by machines alone. However, many concerns cannot or should not be so rigidly defined in code and must be augmented with more subjective judgements. The rules that dictate these subjective constraints to which actors are subjected to are defined through the DAO's agreement, which is part of its configuration from the start. This gives all actors a clear set of rules to which they must abide to engage with the organization and avoid losing collateral.

### Development Environment

The contracts are split in two projects: [`erc3k`](https://github.com/aragon/govern/blob/master/packages/erc3k) (the interfaces defining the ERC3000 standard), and [`govern-core`](https://github.com/aragon/govern/blob/master/packages/govern-core) (the Aragon Govern contracts, implementing ERC3000).

Relevant packages:

- [`erc3k`](https://github.com/aragon/govern/blob/master/packages/erc3k): ERC3000 interfaces.
- [`Govern Core`](https://github.com/aragon/govern/blob/master/packages/govern-core): Aragon ERC3000 implementation.
- [`Govern Create`](https://github.com/aragon/govern/blob/master/packages/govern-create): Set of templates used to create new Govern instances.
- [`Govern Contract Utils`](https://github.com/aragon/govern/blob/master/packages/govern-contract-utils): Set of libraries and utilities used by the Govern contracts.

#### Setup

Start by bootstrapping the entire monorepo with `yarn`:

```text
yarn
```

This will install all needed dependencies, and link all packages together to make sure you're using the local version of each one. After this, we can go and init our local development environment. Go ahead, and use the following command:

```bash
# For this to work, you'll need to have docker installed.
yarn init:dev:env
```

This will, in order:

- Compile all contracts, in the correct order
- Extract all ABIs so the subgraph can reference them properly
- Init a set of containers with an IPFS node, a local Ethereum node \(using Ganache\), and a local instance of the subgraph.

With this, you'll have a local development environment where you can deploy the entire Govern infra, and query the subgraph.

> Right now, all of this is manual; later down the road a more complete development environment with multiple network options \(mainnet fork, and clean local environment with dummy data\) will be made to make test runs easier.

## Console

The Aragon Govern Console is a no-frills, forkable, extensible power user / developer UI tool for interacting with and visualizing low level information about Govern DAOs. Available on [console.aragon.org](https://console.aragon.org).

![The Aragon Govern Console](https://user-images.githubusercontent.com/36158/97722356-77c04900-1ac2-11eb-8a5c-5034a54cdbb4.png)

Relevant package:

- [`Govern Console`](https://github.com/aragon/govern/blob/master/packages/govern-console)

## Govern.js

> [`Govern.js`](https://github.com/aragon/govern/blob/master/packages/govern)

**Usage with default config**

> The default config uses TheGraph nodes hosted by TheGraph Foundation

``` javascript
import { dao } from '@aragon/govern'

const response = await dao('AN-DAO');

console.log(response)
```

**Usage with custom config**
``` javascript
import { dao, configure } from '@aragon/govern'

configure({subgraphURL: 'https://myOwnTheGraphNode.io/aragon-govern'});

const response = await dao('AN-DAO');

console.log(response)
```

---

#### dao(name) ⇒ ``Promise<Dao>``

Returns details about a DAO by his name.

| Param  | Type                  | Description                               |
| ------ | --------------------- | ----------------------------------------- |
| name   | <code>string</code>   | The name of the DAO                       |

Example:
``` typescript 
import { dao } from '@aragon/govern'

const response = await dao('AN-DAO');
```

---

#### daos() ⇒ ``Promise<Dao[]>``

Returns all DAOs.


Example:
``` typescript 
import { daos } from '@aragon/govern'

const response = await daos();
```

---

#### query(query, args) ⇒ ``Promise<object>``

Returns the desired data.

| Param  | Type                      | Description                               |
| ------ | ------------------------- | ----------------------------------------- |
| query  | <code>string</code>       | GraphQL query                             |
| args   | <code>object</code>       | Object with parameters                    |

Example:
``` typescript 
import { query } from '@aragon/govern'

const query = await query(myQuery, {...});
```

---

#### configure(config) ⇒ ``void``

Overwrites the default configuration of govern.

| Param  | Type                                                    | Description                                          |
| ------ | ------------------------------------------------------- | ---------------------------------------------------- |
| config | <code>[ConfigurationObject][ConfigurationObject]</code> | Object with all [config options][ConfigurationObject]|

Example:
``` typescript 
import { configure } from '@aragon/govern'

configure({subgraphUrl: 'https://myOwnTheGraphNode.io/aragon/govern'});
```

[ConfigurationObject]: https://github.com/aragon/govern/tree/master/packages/govern/internal/configuration/Configuration.ts#L4

---

## Subgraph

> [`Govern Subgraph`](https://github.com/aragon/govern/blob/master/packages/govern-subgraph)

### GovernRegistry
``` graphql
type GovernRegistry @entity {
  id: ID!
  address: Bytes!
  count: Int!
  daos: [Dao!]!
}
```

The ``GovernRegistry`` as the name is saying is here to register your DAO within the Govern system.

### DAO
``` graphql
type Dao @entity {
  id: ID!
  name: String!
  queue: GovernQueue!
  executor: Govern!
  token: String!
  registrant: String!
}
```

This GraphQL entitiy holds anything you need to know about your DAO.

### Govern
``` graphql
type Govern @entity {
  id: ID!
  address: Bytes!
  metadata: Bytes
  balance: BigInt!
  roles: [Role!]!
}
```

The ``Govern`` entitiy is the ``Executor`` of ERC3k and can get compared with the Agent from V1 of Aragon.

### GovernQueue
``` graphql
type GovernQueue @entity {
  id: ID!
  address: Bytes!
  nonce: BigInt!
  config: Config!
  containers: [Container!]! @derivedFrom(field: "queue")
  roles: [Role!]!
}
```

The ``GovernQueue`` entitiy is the actual execution delay of the Govern optimistic governance system and contains all scheduled, challenged, and executed containers.
By side of the containers of the queue can you also find the configuration of it with the rules (DAO Agreement) and the collaterals required to interact with.

### Container
``` graphql
type Container @entity {
  id: ID!
  state: ContainerState!
  queue: GovernQueue!
  config: Config!
  payload: Payload!
  history: [ContainerEvent!]! @derivedFrom(field: "container")
}
```

The ``Container`` entity of Govern is the actual execution payload for the queue. It contains the on-chain actions it should execute, the current state of the scheduled execution, and as many additional details you need to inform your user about the exact details of a execution.

### Config
``` graphql
type Config @entity {
  id: ID!
  executionDelay: BigInt!
  scheduleDeposit: Collateral!
  challengeDeposit: Collateral!
  resolver: Bytes!
  rules: Bytes!
}
```

The ``Config`` entity is stored within the ``GovernQueue`` and contains all the important configurations for your Govern based DAO.

### Payload
``` graphql
type Payload @entity {
  id: ID!
  nonce: BigInt!
  executionTime: BigInt!
  submitter: Bytes!
  executor: Govern!
  actions: [Action!]! @derivedFrom(field: "payload")
  allowFailuresMap: Bytes!
  proof: Bytes!
}
```

The ``Payload`` entity contains anything needed for the on-chain execution.

### Collateral
``` graphql
type Collateral @entity {
  id: ID!
  token: Bytes!
  amount: BigInt!
}
```

The ``Collateral`` entity is used within the ``Config`` entitiy to define the token and amount needed to schedule or challenge a container.

### Action
``` graphql
type Action @entity {
  id: ID!
  to: Bytes!
  value: BigInt!
  data: Bytes!
  payload: Payload!
}
```

The ``Action`` entity is used within the ``Payload`` actions array and defines one on-chain call of the container.

### Role
``` graphql
type Role @entity {
  id: ID!
  entity: Bytes!
  selector: Bytes!
  who: Bytes!
  granted: Boolean!
  frozen: Boolean!
}
```

The ``Role`` entitiy is used for the ``ACL`` of Govern.

### ContainerState
``` graphql
enum ContainerState {
  None
  Scheduled
  Approved
  Challenged
  Rejected
  Cancelled
  Executed
}
```
This enumaration is used to define the current state of the ``Container``entity.

### ContainerEvent
``` graphql
interface ContainerEvent {
  id: ID!
  container: Container!
  createdAt: BigInt!
}

type ContainerEventChallenge implements ContainerEvent @entity {
  id: ID!
  container: Container!
  createdAt: BigInt!
  challenger: Bytes!
  collateral: Collateral!
  disputeId: BigInt!
  reason: Bytes!
  resolver: Bytes!
}

type ContainerEventExecute implements ContainerEvent @entity {
  id: ID!
  container: Container!
  createdAt: BigInt!
  execResults: [Bytes!]!
}

type ContainerEventResolve implements ContainerEvent @entity {
  id: ID!
  container: Container!
  createdAt: BigInt!
  approved: Boolean!
}

type ContainerEventRule implements ContainerEvent @entity {
  id: ID!
  container: Container!
  createdAt: BigInt!
  ruling: BigInt!
}

type ContainerEventSchedule implements ContainerEvent @entity {
  id: ID!
  container: Container!
  createdAt: BigInt!
  collateral: Collateral!
}

type ContainerEventVeto implements ContainerEvent @entity {
  id: ID!
  container: Container!
  createdAt: BigInt!
  reason: Bytes!
}
```


### Deployments
#### Mainnet

- [Aragon Govern Mainnet](https://thegraph.com/explorer/subgraph/aragon/aragon-govern-mainnet)

#### Rinkeby

- [Aragon Govern Rinkeby](https://thegraph.com/explorer/subgraph/aragon/aragon-govern-rinkeby)

## Smart Contracts

> ERC3000 specific contracts haven't been included here. To browse them, go to the [corresponding package](https://github.com/aragon/govern/tree/master/packages/erc3k).

**Stateless contracts**

Aragon Govern's contracts hold very little state; this is to keep gas costs as low as possible and to keep the architecture lean. Akin to [Stateless Ethereum](https://blog.ethereum.org/2020/01/28/eth1x-files-the-stateless-ethereum-tech-tree/), to witness all state transitions and ensure all data is forever available for querying and retrieval in an easy manner, we rely on our subgraph, which stores all actions and executions regarding Govern DAOs. Please refer to the [subgraph docs](#subgraph) to have more insight on which events are we tracking, and how.

### GovernRegistry

[📜 Implementation](https://github.com/aragon/govern/blob/master/packages/erc3k/contracts/IERC3000Registry.sol)

The `ERC3000Registry` contract serves as the **central registry for all Govern DAOs**. Every DAO spawned through a properly-implemented `GovernFactory` will register the new DAO in the "official" registry. The registry takes care of doing these things:

* Keep track\* of every `Govern` ⟷ `GovernQueue` pair assigning it a name on the blockchain. This means that instead of assigning an ENS name, this is left to the user, as the name itself will be saved in the contract's storage.
* Setting metadata, which means you can include an IPFS CID to talk about your DAO, and include relevant links.

**Think of it as...**

A book which keeps track of every DAO and its core relevant info, which will be accessible through a subgraph.

{% hint style="info" %}
\* Due to the stateless nature of these contracts, it actually only emits an event, and registers the DAO name to the contract's storage so it cannot be overwritten. We rely on the subgraph to get this information.
{% endhint %}

### GovernFactory & GovernQueueFactory

[🏭 Implementation](https://github.com/aragon/govern/blob/master/packages/govern-create/contracts/GovernBaseFactory.sol)

`GovernFactory` and `GovernQueueFactory` produce their respective contract instances: `Govern` and `GovernQueue`. The intended usage is to create a new GovernQueue, and feed the resulting address, along with an `ERC3000Registry` address to GovernFactory. The latter will then produce a new `Govern` contract, register it to the desired `ERC3000Registry` with the chosen name, and set up a standard set of permissions.

**Think of it as...**

**Templates**; these could be modified and extended to set up more specific permissions and change the collateral needed to execute actions. This would be easy to do even for solidity beginners, considering these contracts don't inherit from other ones, and all permissions are already laid out.

### Govern

[🐣 Implementation](https://github.com/aragon/govern/blob/master/packages/govern-core/contracts/Govern.sol)

`Govern` is the DAO's executor _and_ vault of the organization. It will be responsible for executing the actions that have been scheduled through the queue and holding the organization's funds. While the smart contract is extremely simple \(&lt;80 LOC\), it can effectively call any external smart contract, which means it's basically a smart account which is governed by the DAO.

### GovernQueue

[✨ Implementation](https://github.com/aragon/govern/blob/master/packages/govern-core/contracts/pipelines/GovernQueue.sol)

`GovernQueue` is by far the most critical contract to understand, as it's the main point of interaction with the DAO and the Aragon Protocol. This is what most users will interact with directly—it holds the DAOs configuration parameters, and its where actors can schedule, execute, veto and challenge actions.

`GovernQueue` **can be configured**, meaning you can change these parameters:

```text
struct Config {
  uint256 executionDelay;
  Collateral scheduleDeposit;
  Collateral challengeDeposit;
  address resolver;
  bytes rules;
}

struct Collateral {
  address token;
  uint256 amount;
}
```

The intended workflow for actions would be as follows, assuming the action was sent on chain to the optimistic queue \(note that we're able to do this through `GovernQueue` only\):

1. The action is **scheduled**. **A time window opens** for disputing this action.
   1. If the action is **challenged**, the execution is paused and the action is sent to the arbitrator \(Aragon Protocol\) to resolve the dispute
   2. If it is disputed in favor of the **submitter**, it will release the collateral to the submitter and execute the action\(s\).
   3. If it is disputed in favor of the **challenger**, it will cancel the execution of the action\(s\) and release the collateral to the challenger.
2. The action has successfully passed the time window for challenges, and can be **executed**.
3. Anyone with the necessary permissions calls the `execute` method and `GovernQueue` will make the Govern contract execute the actions.

Actions can be **vetoed**, which might be useful for projects which are venturing into further decentralizing but still desire a multisig for the time being, or want a way for its community to cancel a vote.

### ACL

[🚦Reference implementation](https://github.com/aragon/govern/blob/master/packages/govern-contract-utils/contracts/acl/ACL.sol)

The ACL from govern is a much leaner implementation of the original ACL from aragonOS, but still very powerful, having the ability to grant, revoke, and freeze roles. There are a couple of differences:

* Is devised to be as an **inheritable** contract. Instead of being a single contract that binds the whole organization together, **both `GovernQueue` and `Govern` have their own ACLs**.
* It has a handy **bulk** function to set multiple permissions at once.
* The address for freezing a role is `0x0000000000000000000000000000000000000001`.
* The address for giving the permission to everyone is`0xffffffffffffffffffffffffffffffffffffffff`

### Deployments

Log of deployed govern relevant instances and related infrastructure.

**See [aragon/govern deployments](https://github.com/aragon/govern/tree/master/deployments)**

#### Mainnet

- 📜 GovernRegistry: [`0x9dDC0BAB6aCCa5F374E2C21708b3107e5E973601`](https://etherscan.io/address/0x9dDC0BAB6aCCa5F374E2C21708b3107e5E973601)
- 🏭 GovernBaseFactory: [`0x3B02e7C7Af1be87BBEc071f5DFfcdD8613154bA9`](https://etherscan.io/address/0x3B02e7C7Af1be87BBEc071f5DFfcdD8613154bA9)

#### Rinkeby

- 📜 GovernRegistry: [`0x87eE5EA31dCf1f526f21Bb576131C37890AE65E0`](https://rinkeby.etherscan.io/address/0x87eE5EA31dCf1f526f21Bb576131C37890AE65E0)
- 🏭 GovernBaseFactory: [`0x615e3d83B8e1403c39F98c7066faA1D9bBF9E867`](https://rinkeby.etherscan.io/address/0x615e3d83B8e1403c39F98c7066faA1D9bBF9E867)

## ERC3000

> 📚 Read the formal proposal [here](https://eips.ethereum.org/EIPS/eip-3000).

#### Historical background

Throughout 2020 Ethereum saw the explosion of DeFi, introducing protocols with interesting proposals such as [Balancer](https://balancer.finance/) with programmable liquidity pools and [Yam](https://yam.finance/) with a rebasing mechanism to control supply and widely championing yield farming for everyone. More seasoned players of the space like [Aave](https://aave.com/) brought innovative features like flash loans, but what changed the space was the popularization of _Governance Tokens._

While protocols like Aragon and Maker had already been championing the idea of community governance with tokens, new protocols distributed their token supply through yield farming strategies; those who did attracted high amounts of capital and built healthy treasuries. The tokens could then be used to govern the protocol itself in a community-oriented manner, but one thing became clear: due to the amount of activity on-chain, gas prices arose and DAO activity was quickly priced out. At some point, creating a traditional Moloch or Aragon DAO costed more than **$500**.

Balancer, in response to rising gas prices, built [Snapshot](https://docs.snapshot.page/)—a **gasless multi-governance client** which allowed protocols to do governance votes with no cost, utilizing a snapshot of an user's token balance at a certain block height to measure voting power. This quickly became the de-facto way of doing governance throughout the summer. Of course, this had one pitfall: projects couldn't be fully decentralized, due to the requirement of a community multi-sig to enact off-chain votes on-chain. This is where ERC300 steps in.

#### The standard

ERC-3000 presents a basic on-chain spec for contracts to optimistically enact governance decisions made off-chain. Backed by [Aragon](https://aragon.org) & [Balancer](https://balancer.finance), it's an attempt to standardize the design and writing of governance tools to introduce a high level of composability and compatibility between them.

The standard is opinionated in defining the 6 entrypoint functions to contracts supporting the standard. But it allows for any sort of resolver mechanism for the challenge/response games characteristic of optimistic contracts. 

While the authors \(Jorge Izquierdo from Aragon and Fabien Marino from Balancer\) currently believe resolving challenges [using a subjective oracle](https://aragon.org/blog/snapshot) is the right tradeoff, the standard has been designed such that changing to another mechanism is possible \(a deterministic resolver like [Optimism’s OVM](https://optimism.io) uses\), even allowing to hot-swap it in the same live instance.

