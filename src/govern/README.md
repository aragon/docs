---
sidebar: auto
---

# Govern

![Aragon Govern header](/govern.png)

<p style="text-align: center; font-size: 24px;">Welcome to Govern's documentation.</p>

Govern is software for creating and governing organizations such as DeFi projects, open source projects, gaming guilds, cooperatives, nonprofits, clubs, companies, and any other type of organization you can imagine. It's Aragon's implementation of ERC-3000, the standard for binding off-chain voting.

Along with off-chain voting solutions like [Vocdoni](https://docs.vocdoni.io/) it allows you to _govern_ all parts of your project, even if not completely decentralized, in an easy manner. Also allowing for flexibility and extensions when needed.

If you have any questions or you'd like to say hi, come join the community at [Discord](https://discord.com/invite/aragon)!

## Introduction

### Anatomy of a Govern DAO

A Govern DAO, in its most normal and basic form, consists two key parts:

* An Action Queue
* An Executor

Govern's Action Queue is what most users will interact with directly. It holds the DAO's configuration parameters, and allows actors to schedule, execute, challenge, and veto actions. It's also flexible, meaning that actions can be introduced to tweak its own parameters on the fly.

Through the Executor, Govern DAOs can also hold funds and interact with any arbitrary protocol. This makes it equivalent to **aragonOS's Agent** and **Moloch's Minion.** This allows for complex user flows. For example, the community decides, through a Vocdoni's voting solution, to use pooled funds in the Executor. With this, the community aims to gain interest through lending protocols (like Compound or AAVE). Additionally, they decide to fund a Balancer liquidity pool. All of this can happen in one transaction.

Learn more about the [smart contract system here](#smart-contracts).

Flow-Chart of Govern:
![Aragon Govern Flow-Chart](/govern_flow_chart.png)

Govern will also provide first-class support for Vocdoni and Snapshot. A _space_ or _token_ will be configurable, such that every proposal can attach an on-chain action to the vote's options. With these tools, we have everything that is needed for a project to decide its future in a community-driven manner.

### Optimistic by design

Govern builds on the "Optimistic" concept, in which we assume actors are acting rationally and with good intentions - until proven otherwise. Actors with permissions in the DAO can submit actions, which can get executed after a defined time window. These actions can also be disputed with the chosen mechanism for resolving these challenge/response games. In this case, this will most likely be a [subjective oracle](https://aragon.org/blog/snapshot), such as Aragon Court. Actions can also be vetoed by an actor with the corresponding permissions, as long as they haven't been executed.

With the shift towards operating a DAO in an optimistic way, economic and social incentives are introduced for people to act rationally: any malicious actions that get submitted can get challenged. This results in the malicious actor losing money and the challenger earning the actor's collateral. However, unnecessary challenges are also disincentivized, as they require to lock a collateral. Challenges, therefore, come with a potential loss of reputation, if a subjective oracle rules in favor of the submitter.

To date, acceptable actions and behavioral boundaries within DAOs are rigidly defined in code, and strictly enforced by machines alone. However, many concerns cannot (or should not) be defined so rigidly in code, and must be augmented with more subjective judgements. The rules that dictate the subjective constraints to which actors are subjected, are defined through the DAO's agreement. The agreement is part of the DAO's configuration from the start. This gives all actors a clear set of rules by which they must abide if they want to engage with the organization and want to avoid losing collateral.

### Development Environment

The contracts are split in two projects: [`erc3k`](https://github.com/aragon/govern/blob/master/packages/erc3k) (the interfaces defining the ERC3000 standard), and [`govern-core`](https://github.com/aragon/govern/blob/master/packages/govern-core) (the Aragon Govern contracts, implementing ERC3000).

Relevant packages:

- [`erc3k`](https://github.com/aragon/govern/blob/master/packages/erc3k): ERC3000 interfaces.
- [`Govern Core`](https://github.com/aragon/govern/blob/master/packages/govern-core): Aragon's ERC3000 implementation.
- [`Govern Create`](https://github.com/aragon/govern/blob/master/packages/govern-create): Set of templates used to create new Govern instances.
- [`Govern Contract Utils`](https://github.com/aragon/govern/blob/master/packages/govern-contract-utils): Set of libraries and utilities used by the Govern contracts.

#### Setup

Start by bootstrapping the entire monorepo with `yarn`:

```text
yarn
```

This will install all required dependencies, and link all packages together to make sure you're using the local version of each one. After this, you can initialize your local development environment. Go ahead, and use the following command:

```bash
# For this to work, you'll need to have docker installed.
yarn init:dev:env
```

This will, in order:

- Compile all contracts in the correct order
- Extract all ABIs so the subgraph can reference them properly
- Init a set of containers with an IPFS node, a local Ethereum node \(using Ganache\), and a local instance of the subgraph.

With this, you'll have a local development environment where you can deploy the entire Govern infra, and query the subgraph.

> Right now, all of this is done manually. Down the road, we aim to provide a more complete development environment. It will include multiple network options \(mainnet fork, and clean local environment with dummy data\), which will make test-runs easier.

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

Returns details about a DAO by its name.

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

Overwrites the default configuration of Govern.

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

The ``GovernRegistry`` registers your DAO within the Govern system.

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

The ``Dao`` entitiy holds everything you need to know about your DAO.

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

The ``Govern`` entity is the ``Executor`` of ERC3k and is comparable to the Agent in Aragon V1.

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

The ``GovernQueue`` entity performs the execution delay for Govern's optimistic governance system. It contains all scheduled, challenged, and executed containers.
Along with the containers in the queue, you can also find their configuration, rules (DAO Agreement) and collaterals required to interact with them.

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

The ``Container`` entity of Govern is the execution payload for the queue. It contains the on-chain actions it should execute, the current state of the scheduled execution. Additionally, it can hold any number of additional details you need to inform your user about an execution.

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

The ``Config`` entity is stored within the ``GovernQueue`` and contains all the important configurations for your Govern-based DAO.

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

The ``Payload`` entity contains all the information necessary for on-chain execution.

### Collateral
``` graphql
type Collateral @entity {
  id: ID!
  token: Bytes!
  amount: BigInt!
}
```

The ``Collateral`` entity is used within the ``Config`` entity to define the type and amount of token needed to schedule or challenge a container.

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

The ``Action`` entity is used within the ``Payload``'s actions array and defines one on-chain call of the container.

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

The ``Role`` entity is used for the ``ACL`` of Govern.

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
This enumaration is used to define the current state of the ``Container`` entity.

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

Aragon Govern contracts hold very little state; this is done to keep gas costs as low as possible and to keep the architecture lean. Akin to [Stateless Ethereum](https://blog.ethereum.org/2020/01/28/eth1x-files-the-stateless-ethereum-tech-tree/), to witness all state transitions and ensure tha all data is forever available for querying and retrieval in an easy manner, we rely on our subgraph. It stores all actions and executions regarding Govern DAOs. Please refer to the [subgraph docs](#subgraph) to have more insight on which events are we tracking, and how.

### GovernRegistry

[📜 Implementation](https://github.com/aragon/govern/blob/master/packages/erc3k/contracts/IERC3000Registry.sol)

The `ERC3000Registry` contract serves as the **central registry for all Govern DAOs**. Every DAO spawned through a properly-implemented `GovernFactory` will register the new DAO in the "official" registry. The registry takes care of doing the following:

* Keep track\* of every `Govern` ⟷ `GovernQueue` pair, assigning it a name on the blockchain. This means that instead of assigning an ENS name, this is left to the user, as the name itself will be saved in the contract's storage.
* Setting metadata. This means that you can include an IPFS CID to provide further information about your DAO, and include relevant links.

**Think of it as...**

A book that keeps track of every DAO and its core information, accessible through a subgraph.

{% hint style="info" %}
\* Due to the stateless nature of these contracts, it actually only emits an event and registers the DAO name in the contract's storage so it cannot be overwritten. We rely on the subgraph to get this information.
{% endhint %}

### GovernFactory & GovernQueueFactory

[🏭 Implementation](https://github.com/aragon/govern/blob/master/packages/govern-create/contracts/GovernBaseFactory.sol)

`GovernFactory` and `GovernQueueFactory` produce their respective contract instances: `Govern` and `GovernQueue`. The intended way of using them is to 

* create a new GovernQueue
* feed the resulting address, along with an `ERC3000Registry` address to GovernFactory. 

The GovernFactory will then produce a new `Govern` contract, register it to the desired `ERC3000Registry` with the chosen name, and set up a standard set of permissions.

**Think of it as...**

**Templates** that can be modified and extended to set up more specific permissions and change the collateral needed to execute actions. This should be feasible for solidity beginners, considering these contracts don't inherit from other ones, and all permissions are already laid out.

### Govern

[🐣 Implementation](https://github.com/aragon/govern/blob/master/packages/govern-core/contracts/Govern.sol)

`Govern` is the DAO's executor _and_ vault of the organization. It will be responsible for executing the actions that have been scheduled through the queue and holding the organization's funds. Altough the smart contract is extremely simple \(&lt;80 LOC\), it can effectively call any external smart contract. This basically makes it a _smart account_, which is governed by the DAO.

### GovernQueue

[✨ Implementation](https://github.com/aragon/govern/blob/master/packages/govern-core/contracts/pipelines/GovernQueue.sol)

`GovernQueue` is by far the most critical contract to understand, as it's the main point of interaction with the DAO and the Aragon Protocol. This is what most users will interact with directly. It holds the DAO's configuration parameters, and allows actors to schedule, execute, challenge, and veto actions.

`GovernQueue` **can be configured**, meaning you can change the following parameters:

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
   1. If the action is **challenged**, the execution is paused, a collateral is locked, and the action is sent to the arbitrator \(Aragon Protocol\) to resolve the dispute.
   2. If it is disputed in favor of the **submitter**, it will release the collateral to the submitter and execute the action\(s\).
   3. If it is disputed in favor of the **challenger**, it will cancel the execution of the action\(s\) and release the collateral to the challenger.
2. The action has successfully passed the time window for challenges and can be **executed**.
3. A user with the necessary permissions calls the `execute` method and `GovernQueue` will make the Govern contract execute the action.

Actions can be **vetoed**. This might be useful for projects that are venturing into further decentralization, but still desire a multisig for the time being, or want a way for its community to cancel a vote.

### ACL

[🚦Reference implementation](https://github.com/aragon/govern/blob/master/packages/govern-contract-utils/contracts/acl/ACL.sol)

The ACL from govern is a much leaner implementation of the original ACL from aragonOS. However, it still very powerful, having the ability to grant, revoke, and freeze roles. There are a couple of differences:

* ACL is devised to be an **inheritable** contract. Instead of being a single contract that binds the whole organization together, **both `GovernQueue` and `Govern` have their own ACLs**.
* ACL has a handy **bulk** function to set multiple permissions at once.
* The address for freezing a role is `0x0000000000000000000000000000000000000001`.
* The address for giving the permission to everyone is`0xffffffffffffffffffffffffffffffffffffffff`

### Deployments

Log of relevant deployed govern instances and related infrastructure.

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

While protocols like Aragon and Maker had already been championing the idea of community governance with tokens, new protocols distributed their token supply through yield farming strategies; those who did attracted high amounts of capital and built healthy treasuries. The tokens could then be used to govern the protocol itself in a community-oriented manner. On the flip-side, this lead to increased on-chain activity. In turn, gas prices rose and DAO activity was quickly priced out. At some point, creating a traditional Moloch or Aragon DAO costed more than **$500**.

Balancer, in response to rising gas prices, built [Snapshot](https://docs.snapshot.page/)-a **gasless multi-governance client**, which allows protocols to perform governance votes at no cost. It works by utilizing a snapshot of a user's token balance at a certain block height to measure voting power. This quickly became the de-facto way of doing governance throughout the summer. Of course, this has an important caveat: projects couldn't be fully decentralized, due to the requirement of a community multi-sig to enact off-chain votes on-chain. This is where ERC3000 steps in.

#### The standard

ERC-3000 presents a basic on-chain spec for contracts to optimistically enact governance decisions made off-chain. Backed by [Aragon](https://aragon.org) & [Balancer](https://balancer.finance), it attempts to standardize the design and writing of governance tools, and to introduce a high level of composability and compatibility between them.

The standard is opinionated in defining the 6 entrypoint functions to contracts supporting the standard. But it allows for any sort of resolver mechanism for the challenge/response games characteristic of optimistic contracts. 

While the authors \(Jorge Izquierdo from Aragon and Fabien Marino from Balancer\) currently believe resolving challenges [using a subjective oracle](https://aragon.org/blog/snapshot) is the right tradeoff, the standard has been designed such that changing to another mechanism is possible \(a deterministic resolver like [Optimism’s OVM](https://optimism.io) uses\), even allowing to hot-swap it in the same live instance.
