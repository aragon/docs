---
sidebar: auto
---

# Court

The following documents attempt to be a high-level description of the inner implementation details of Aragon Court. It doesn't go deep to the point of being an exhaustive spec detailing all authentication checks and state transitions, but aims to provide enough description for a developer to deeply understand the existing codebase or guide a re-implementation.

This document was written to ease the job of security auditors looking at the codebase and was written after the v1 implementation had been frozen.

The core of the document is organized around the external entry points to the system across the different modules.

## Mechanism

The Aragon Protocol is a dispute resolution protocol designed to handle subjective disputes which cannot be arbitrated by smart contracts. At a high level, this is achieved by drafting a random set of guardians for each dispute over which a ruling is voted over. Aragon Protocol is one of the core components of the [Aragon Network](https://aragon.org/network/).

Ethereum accounts (including contracts) will sign up to be guardians by staking ANT tokens into the Protocol. The more tokens a guardian has staked and activated, the higher their chance of being drafted.

Based on the concept of a [Schelling point](https://en.wikipedia.org/wiki/Focal_point_(game_theory)), guardians are asked to vote on the ruling they think their fellow guardians are most likely to vote on. Every time a guardian is drafted for a dispute, a portion of their active tokens are locked until the dispute is finalized. To incentivize consensus, guardians that don’t vote in favor of the consensus ruling have their locked tokens slashed. Guardians that vote in favor of the consensus ruling are rewarded with ruling fees and a portion of the tokens slashed from the minority-voting guardians.

Once a ruling has been decided for a dispute, there is a time period where anyone is allowed to appeal said ruling by putting some collateral at stake to initiate a new dispute round. If this occurs, a new set of guardians will be drafted and a new ruling will be voted on. Rulings can be appealed multiple times until the final round is reached. To mitigate 51% attacks, all active guardian accounts can opt into voting during the final round. For future versions of the Protocol, we are considering using [futarchy decision markets](https://blog.aragon.one/futarchy-protocols/) for the final dispute round instead.

The Protocol uses an inherent time unit, called a "term," to determine how long certain actions or phases last. The length of a "term" is guaranteed to be constant, although each phase in a dispute can have its length configured by setting how many "terms" the phase will last for. Terms are advanced via "heartbeat" transactions and most functionality will only execute if the Protocol has been updated to its latest term.

Even though the Aragon Protocol could theoretically resolve any type of binary dispute, we intend for its first version to be primarily used for arbitrating [Proposal Agreements](https://blog.aragon.one/proposal-agreements-and-the-aragon-protocol/). These agreements require entities to first agree upon a set of rules and processes for creating proposals in an organization, each forcing proposal creators to stake collateral that may be forfeit if the proposal is deemed invalid by the Protocol. However, how disputes arrive at the Aragon Protocol is outside of the scope of this protocol. The Protocol relies on a small external interface to link corresponding disputes and execute them once a ruling has been decided.

### High-level flow

- Guardians stake ANT to the Protocol contract and schedule their activation and deactivation for the time period in which they can be drafted to rule on disputes.
- Protocol fees and configuration parameters are controlled by a governor (eventually the Aragon Network), but can only be modified for future terms to ensure that parameters can’t change for ongoing disputes.
- The creator of a dispute must pay fees to cover the maintenance gas costs of the Protocol and the guardians that will adjudicate their dispute. The governor of the Protocol receives a share of all fees paid to the Protocol.
- Guardians are randomly drafted to adjudicate disputes, where the chance to be drafted is proportional to the amount of ANT they have activated.
- When drafted, each guardian must commit and reveal their vote for the ruling. Failure to commit or reveal results in a penalty for the guardian.
- After a ruling is decided, it can be appealed by anyone a certain number of times, after which all active guardians will vote on the last appeal (an unappealable ruling).
- When the final ruling is decided, all the adjudication rounds for the dispute can be settled, taking into account the final ruling for rewards and penalties.

## Architecture

![Architecture diagram](/court_architecture.png)

### Controller

The `Controller` has four main responsibilities:
- Permissions management
- Modules management
- Protocol terms ("clock") management
- Protocol configuration management

The protocol relies on five main modules: `DisputeManager`, `Voting`, `GuardiansRegistry`, `Treasury`, and `PaymentsBook`.
Each of these modules are only referenced by the `Controller`; centralizing them allows us to be able to plug or unplug modules easily.

The Protocol terms management and reference is held in `Clock`. Almost every functionality of the protocol needs to ensure the current Protocol term is up-to-date.

Every protocol configuration variable that needs to be check-pointed based on the different terms of the Protocol is referenced in `Config`.
Note that this is important to be able to guarantee to our users that certain actions they committed to the Protocol will rely always on the same configurations of the protocol that were there at the moment said actions were requested.
On the other hand, there are some configuration variables that are related to instantaneous actions. In this case, since we don't need to ensure historic information, these are held on its corresponding module for gas-optimization reasons.

### Modules

All the modules of the protocol can be switched by new deployed ones. This allows us to have recoverability in case there is a failure in any of the modules.
Each module has a reference to the `Controller` through `Controlled`, and `ControlledRecoverable` is a special flavor of `Controlled` to ensure complete recoverability.
Since some modules handle token assets, we need to be able to move these assets from one module to another in case they are switched.

However, these modules can be frozen at any time, once the protocol has reached a high-level of maturity full-immutability will be guaranteed.
In case further changes are required a new whole implementation of the protocol can be deployed and it will be a users decision to chose the one they want to use.

Detailed information about each module functionality is described in [section 4](../4-entry-points).

### Permissions

The permissions layer is self-controlled in the `Controller`. This component works with three `Governor` addresses to implement that:
- Modules governor
- Config governor
- Funds governor

All the functionality described above whose access needs to be restricted relies on these addresses.
The modules governor is the only one allowed to switch modules. This address can be unset at any time to ensure no further changes on the modules can be made.
Finally the funds governor is the only one allowed to recover assets from the `ControlledRecoverable` modules. As for the modules governor, this address can be unset at any time.
The config governor is the only one allowed to change all the configuration variables of the protocol. This last one, is the only governor address that is not intended to be unset.
The protocol settings should be always able to be tweaked to ensure the proper operation of Protocol. To guarantee decentralized governance of these variables, the config governor is meant to be the Aragon Network.

Any other modules functionality that needs to be restricted is either relying on these governor addresses as well, or on any of the other modules.
No other external accounts apart from the `Governor` addresses belong to the protocol implementation.

As described in the [launch process](https://forum.aragon.org/t/aragon-network-launch-phases-and-target-dates/1263) of Aragon Protocol, the governor will be a DAO properly called Aragon Network DAO managed by ANT holders.
Initially, the Aragon Network DAO will be governed by a small council with a group trusted members from the community and will be transitioned to the ANT holders at the end of the launch process.

### Entry point

The main entry point of the protocol is `AragonProtocol`, this component inherits from `Controller`.
This allows to guarantee a single and immutable address to the users of the protocol. `AragonProtocol` does not implement core logic, only the main entry points of the protocol where each request is forwarded to the corresponding modules of the `Controller` to be fulfilled.

Detailed information about `AragonProtocol` can be found in [section 4](../4-entry-points).

### Migration Strategies

| Module             |                          Ongoing disputes                         |                              No disputes                          |
|--------------------|-------------------------------------------------------------------|-------------------------------------------------------------------|
| Dispute Manager    | Leave the previous instance active until all disputes are solved  | Disable previous instance and update Voting linked module         |
| Voting             | Deploy new Dispute Manager and point to new Voting instance (*)   | Disable previous instance and update Dispute Manager linked module|
| Treasury           | Deploy new Dispute Manager and point to new Treasury instance (*) | "                                                                 |
| PaymentsBook       | Leave the previous instance active until all funds are claimed                                                                        |
| Guardians Registry | Disable disputes and staking while the status is migrated the the new instance, update Dispute Manager and PaymentsBook modules (**)  |

(*) Assuming there is no easy way to migrate/replicate the current status of the existing module to the new one

(**) If the Guardians Registry status cannot be migrated it would be similar to deploying a new entire protocol

#### Dispute Manager

| Relies on module    |           On           |    Expected behavior      |
|---------------------|------------------------|---------------------------|
| Treasury            | `createDispute`        | The Dispute Manager needs to access the same Treasury instance during a dispute lifecycle |
| Treasury            | `createAppeal`         | " |
| Treasury            | `confirmAppeal`        | " |
| Treasury            | `draft`                | " |
| Treasury            | `settleReward`         | " |
| Treasury            | `settlePenalties`      | " |
| Treasury            | `settleAppealDeposit`  | " |
| Voting              | `createDispute`        | The Dispute Manager needs to access the same Voting instance during a dispute lifecycle |
| Voting              | `ensureCanCommit`      | " |
| Voting              | `createAppeal`         | " |
| Voting              | `confirmAppeal`        | " |
| Voting              | `settleReward`         | " |
| Voting              | `settlePenalties`      | " |
| GuardiansRegistry   | `draft`                | The Dispute Manager needs to access the same Guardians Registry instance during a dispute lifecycle |
| GuardiansRegistry   | `ensureCanCommit`      | " |
| GuardiansRegistry   | `getGuardian`          | " |
| GuardiansRegistry   | `createAppeal`         | " |
| GuardiansRegistry   | `confirmAppeal`        | " |
| GuardiansRegistry   | `settleReward`         | " |
| GuardiansRegistry   | `settlePenalties`      | " |
| GuardiansRegistry   | `settleAppealDeposit`  | " |

Notes:
- If there are no ongoing disputes any of the three dependencies can be swapped
- If there are ongoing disputes none of the three dependencies can be migrated


#### Voting

| Relies on module |           On           |     Expected behavior     |
|------------------|------------------------|---------------------------|
| DisputeManager   | `create`               | Only the latest Dispute Manager should be able to create votes                              |
| DisputeManager   | `commit`               | The Voting module needs to access the same Dispute Manager instance during a vote lifecycle |
| DisputeManager   | `reveal`               | " |
| DisputeManager   | `leak`                 | " |

Notes:
- If there are ongoing disputes the Voting dependency cannot be changed
- If a new Dispute Manager is deployed it can be pointed to the same Voting module


#### Treasury

| Relies on module |           On           |     Expected behavior     |
|------------------|------------------------|---------------------------|
| DisputesManager  | `assign`               | Any active Dispute Manager should be able to assign token balances |

Notes:
- The Treasury module shouldn't reference any particular Dispute Manager instance, deploying a new Dispute Manager is not a problem


#### Guardians Registry

| Relies on module |           On           |    Expected behavior     |
|------------------|------------------------|--------------------------|
| DisputesManager  | `assignTokens`         | Any active Dispute Manager should be able to manage guardian tokens |
| DisputesManager  | `burnTokens`           | " |
| DisputesManager  | `draft`                | " |
| DisputesManager  | `clashOrUnlock`        | " |
| DisputesManager  | `collectTokens`        | " |
| DisputesManager  | `lockWithdrawals`      | " |

Notes:
- The Guardians Registry shouldn't reference any particular Dispute Manager instance, deploying a new Dispute Manager is not a problem


#### PaymentsBook

| Relies on module    |           On           |    Expected behavior     |
|---------------------|------------------------|--------------------------|
| GuardiansRegistry   | `getGuardian`          | Each period should work always with the same Guardians Registry instance |
| GuardiansRegistry   | `getGuardianShare`     | " |
| GuardiansRegistry   | `claimGuardianShare`   | " |
| GuardiansRegistry   | `ensurePeriodBalance`  | " |

Notes:
- The PaymentsBook module could checkpoint the Guardians Registry instance per period
- Otherwise, it can trust that the Guardians Registry always reflects the same status among all its versions

## Crypto-economic considerations

The present implementation is the first iteration of a crypto-economic protocol designed for reaching consensus over subjective issues which can cause side effects in a purely objective deterministic system, such as a public blockchain.

By definition, it is impossible for Aragon Protocol's rulings to be perceived as correct by every observer, as their own personal biases and values can make them interpret a subjective issue in a different way than the majority of guardians.

It's important that all system participants understand these three concepts and take these into account when interacting with the protocol:

### Aragon Court as a Schelling game

Opposed to judges in the legacy legal systems of land jurisdictions, Aragon Protocol guardians are not asked for their impartial opinion on a dispute, but to vote on the ruling they think their fellow guardians will also vote for. Those who vote with the majority are rewarded with tokens that are slashed with guardians that vote for a losing ruling.

If the protocol asked for everyone's unbiased opinion, it would be unfair to penalize those in the minority, as they could have done a perfectly fine job and just not agree with the rest. But if you don't slash those in the minority, attacks would be free and therefore the system wouldn't be secure.

Aragon Protocol is therefore a Schelling game. An easy example of a Schelling game would be if two people are in New York City and they have to meet on a certain day but have no way to coordinate, both would probably pick a recognizable place such as Times Square and go there at noon, as it is the most plausible rational decision making process the other person could do as well.

### Aragon Court as a 'proof of stake' system

When designing a permissionless protocol, that is one in which any participants can come and go without asking anyone's authorization, one has to assume that any given entity can have multiple identities at the same time.

Aragon Protocol was designed with this in mind and assumes participants can [Sybil 'attack](https://www.geeksforgeeks.org/sybil-attack/)' the system without it being an issue for the integrity of the protocol. The amount of active tokens of Aragon Protocol's native token (ANT for Aragon Network's deployment) act as the weight for someone's impact. If someone has 50 ANT, it should be equally or more preferable to have all the tokens active under in one identity, than say 10 ANT across five different identities.

As in most subjective systems in which consensus decisions are weighted by stake, the integrity of the system can be attacked by a cartel with influence over the decisions of more than 50% of the active stake.

From this we can infer that the security of Aragon Protocol is a function of the market capitalization of its native token. A naive calculation would be to assume that one can attack the integrity of the Protocol by spending 51% of the token's market cap to acquire a majority position. In reality this wouldn't be the case, as acquiring 51% of the tokens of a network will almost always be more expensive than 51% of the initial market cap, specially if there's low liquidity and a fair amount of the supply is already staked and active by honest participants.

Even with this in mind, Aragon Protocol has a mechanism to protect an honest minority of guardians from a hostile takeover in the form of a 51% attack: after a certain number of appeal rounds (in which the number of guardians increases geometrically, and therefore the amount of token at stake), there's a final appeal round that will decide the final ruling for the dispute. In the final appeal round, all active guardians are invited to rule, and the final ruling is decided by a majority commit and reveal vote in which participating guardians put all their tokens at stake.

After the final appeal round votes are tallied, a final ruling is issued and every guardian who participated in the dispute and didn't support the winning ruling is slashed. In the case of a successful 51% attack, the attackers will be rewarded with some tokens from honest guardians. Aragon Protocol will then block withdrawals for every guardian that voted for the winning ruling in the final appeal round. The rationale behind this is that even if Aragon Protocol was successfully attacked, malicious guardians will be locked in for a period of time, allowing honest guardians to exit the system and sell their guardian tokens before the attackers are allowed to withdraw. Therefore, the attack is disincentivized with the threat of losing almost all of the value spent purchasing the tokens.

In conclusion, the security of Aragon Protocol is a function of its market cap, although acquiring a majority stake requires an investment more expensive than half of the initial market cap. By locking withdrawals for winning guardians in final appeal rounds, attackers should expect to lose most of the value spent in acquiring their token position, therefore requiring them to pay an exorbitant amount of money to influence the outcome of just one dispute.

### Aragon Protocol as an efficient consensus reaching mechanism

Aragon Protocol tries to reach consensus over a ruling involving the minimum number of participating guardians. The optimal number of participants involved needs to tend to zero, that is, everyone's trust in Aragon Protocol is so high, that misbehaving is disincentivized by the very threat of the Protocol's existence.

When a dispute is created in Aragon Protocol, a sortition process is performed to select the guardians that will adjudicate the dispute. A small random set of guardians is drafted from the active guardian pool (weighted by active stake for sybil resistance) and asked to rule on the dispute. This initial number of guardians can be as small as 3 guardians (or even just 1) and they are expected to provide the ruling that a majority vote among all active guardians would have produced.

The advantage of this is reducing the number of participants that need to review the case and evidence, resulting a more efficient and cheaper system. Because only a small number of participants is involved, any observer can appeal a decision and make a profit if the ruling ends up being flipped. Every appeal geometrically increases the number of guardians drafted to adjudicate the next round, and after a certain number of appeals, there's a final majority vote in which all active guardians can vote to produce the final ruling.

If guardians were only rewarded when drafted to work, by doing their job really well and maintaining a super honest protocol, they would be decreasing their future returns, as the incentive to create disputes in a highly effective protocol is low, as both parties to the dispute can predict what the outcome will be. If guardians leave the Protocol because the returns are decreasing as a result of a well functioning Protocol, the market cap of the token will decrease, making attacks cheaper.

For this reason, Aragon Protocol supports charging payment fees to its users for the right to use the Protocol should a dispute arise. These payment fees are then distributed to all active guardians during that period proportional to their relative stake. Even if no disputes are created in a period of time, guardians should expect a predictable income coming from passive users of the Protocol who are getting security from it even if they don't have the need create disputes.

### A glimpse into Aragon Protocol v2 potential improvements

Even though we consider the system production ready, this is just the first iteration of the protocol in which we have optimized for simplicity of the mechanism. We plan on working on a future version of the protocol taking into account user feedback and tackling some aspects to make the protocol even more robust and making some attacks even more expensive.

At the moment, a ruling can be appealed all the way up to a majority vote happens in the final appeal round. As explained above, this is vulnerable to 51% attacks, even if made really expensive and discouraged by locking winning guardians for a period of time. We did some research on better finality mechanisms and we settled on using futarchy for making the final decision after all appeal rounds occur. By using futarchy, a prediction market can be asked which ruling will make the Protocol be more valuable at some point in the future. This mechanism is not possible to 51% attack, and attackers can expect to lose all their money if honest participants take on the other side of the market to make a profit. You can read a more in-depth description in Luke Duncan's ['Futarchy Protocols' post](https://blog.aragon.one/futarchy-protocols/).

The drafting mechanism currently uses the hash of a future Ethereum block as its randomness seed. Even though the hash of a future block is impossible to predict, the potential miner for a block that will impact a dispute's randomness can decide to drop the block (by not broadcasting it to the network) if its hash is not favorable to them. The miner has the ability to have another drafting chance at the expense of the lost block reward. Given the possibility to use the appeals process all the way up to the entire active guardian set, this vulnerability was considered low risk and decided on using block hash randomness for its simplicity. For a future versions of Aragon Protocol we are exploring using more robust randomness such as a RANDAO mechanism or Keep's Random Beacon.

## Public API

The following sections aim to deeply describe the functionality exposed by each of the components of the protocol mentioned in [section 2](../2-architecture).

### AragonCourt

`AragonProtocol` is the main entry point of the whole protocol and is only responsible for providing a few entry points to the users of the protocol while orchestrating the rest of the modules to fulfill these request.
Additionally, as shown in [section 2](../2-architecture), `AragonProtocol` inherits from `Controller`. The inherited functionality is core to architecture of the protocol and can be found in the [next section](./2-controller.md).
To read more information about its responsibilities and how the whole architecture structure looks like, go to [section 2](../2-architecture).

#### Constructor

- **Actor:** Deployer account
- **Inputs:**
    - **Term duration:** Duration in seconds per Protocol term
    - **First-term start time:** Timestamp in seconds when the Protocol will start
    - **Governor:** Object containing
        - **Funds governor:** Address of the governor allowed to manipulate module's funds
        - **Config governor:** Address of the governor allowed to manipulate protocol settings
        - **Modules governor:** Address of the governor allowed to manipulate module's addresses
    - **Settings:** Object containing
        - **Fee token:** Address of the token contract that is used to pay for the fees
        - **Guardian fee:** Amount of fee tokens paid per drafted guardian per dispute
        - **Heartbeat fee:** Amount of fee tokens per dispute to cover terms update costs
        - **Draft fee:**  Amount of fee tokens per guardian to cover the drafting costs
        - **Settle fee:** Amount of fee tokens per guardian to cover round settlement costs
        - **Evidence terms:** Max submitting evidence period duration in Protocol terms
        - **Commit terms:** Duration of the commit phase in Protocol terms
        - **Reveal terms:** Duration of the reveal phase in Protocol terms
        - **Appeal terms:** Duration of the appeal phase in Protocol terms
        - **Appeal confirmation terms:** Duration of the appeal confirmation phase in Protocol terms
        - **Penalty permyriad:** ‱ of min active tokens balance to be locked for each drafted guardian (1/10,000)
        - **Final-round reduction:** ‱ of fee reduction for the last appeal round (1/10,000)
        - **First-round guardians number:** Number of guardians to be drafted for the first round of a dispute
        - **Appeal step factor:** Increasing factor for the number of guardians of each dispute round
        - **Max regular appeal rounds:** Number of regular appeal rounds before the final round is triggered
        - **Final round lock terms:** Number of terms that a coherent guardian in a final round is disallowed to withdraw
        - **Appeal collateral factor:** ‱ multiple of dispute fees (guardians, draft, and settlements) required to appeal a preliminary ruling (1/10,000)
        - **Appeal confirmation collateral factor:** ‱ multiple of dispute fees (guardians, draft, and settlements) required to confirm an appeal (1/10,000)
        - **Min active balance:** Minimum amount of guardian tokens that can be activated
- **Authentication:** Open
- **Pre-flight checks:** None
- **State transitions:**
    - Call `Controller` constructor

#### Create dispute

- **Actor:** Arbitrable instances, entities that need a dispute adjudicated
- **Inputs:**
    - **Possible rulings:** Number of possible results for a dispute
    - **Metadata:** Optional metadata that can be used to provide additional information on the dispute to be created
- **Authentication:** Open. Implicitly, only smart contracts that have open an ERC20 allowance with an amount of at least the dispute fee to the `DisputeManager` module can call this function
- **Pre-flight checks:**
    - Ensure that the msg.sender supports the `IArbitrable` interface
- **State transitions:**
    - Create a new dispute object in the DisputeManager module

#### Close evidence period

- **Actor:** Arbitrable instances, entities that need a dispute adjudicated.
- **Inputs:**
    - **Dispute ID:** Dispute identification number
- **Authentication:** Open. Implicitly, only the Arbitrable instance related to the given dispute
- **Pre-flight checks:**
    - Ensure a dispute object with that ID exists
    - Ensure that the dispute subject is the Arbitrable calling the function
    - Ensure that the dispute evidence period is still open
- **State transitions:**
    - Update the dispute to allow being drafted immediately

#### Execute dispute

- **Actor:** External entity incentivized to execute the final ruling decided for a dispute. Alternatively, an altruistic entity to make sure the dispute is ruled.
- **Inputs:**
    - **Dispute ID:** Dispute identification number
- **Authentication:** Open
- **Pre-flight checks:**
    - Ensure a dispute object with that ID exists
    - Ensure that the dispute has not been executed yet
    - Ensure that the dispute's last round adjudication phase has ended
- **State transitions:**
    - Compute the final ruling in the DisputeManager module
    - Execute the `IArbitrable` instance linked to the dispute based on the decided ruling

### Controller

The `Controller` is core component of the architecture whose main responsibilities are permissions, modules, Protocol terms, and Protocol configurations management.
To read more information about its responsibilities and structure, go to [section 2](../2-architecture).

#### Constructor

- **Actor:** Deployer account
- **Inputs:**
    - **Term duration:** Duration in seconds per Protocol term
    - **First-term start time:** Timestamp in seconds when the Protocol will start
    - **Governor:** Object containing
        - **Funds governor:** Address of the governor allowed to manipulate module's funds
        - **Config governor:** Address of the governor allowed to manipulate protocol settings
        - **Modules governor:** Address of the governor allowed to manipulate module's addresses
    - **Settings:** Object containing
        - **Fee token:** Address of the token contract that is used to pay for the fees
        - **Guardian fee:** Amount of fee tokens paid per drafted guardian per dispute
        - **Draft fee:**  Amount of fee tokens per guardian to cover the drafting costs
        - **Settle fee:** Amount of fee tokens per guardian to cover round settlement costs
        - **Evidence terms:** Max submitting evidence period duration in Protocol terms
        - **Commit terms:** Duration of the commit phase in Protocol terms
        - **Reveal terms:** Duration of the reveal phase in Protocol terms
        - **Appeal terms:** Duration of the appeal phase in Protocol terms
        - **Appeal confirmation terms:** Duration of the appeal confirmation phase in Protocol terms
        - **Penalty permyriad:** ‱ of min active tokens balance to be locked for each drafted guardian (1/10,000)
        - **Final-round reduction:** ‱ of fee reduction for the last appeal round (1/10,000)
        - **First-round guardians number:** Number of guardians to be drafted for the first round of a dispute
        - **Appeal step factor:** Increasing factor for the number of guardians of each dispute round
        - **Max regular appeal rounds:** Number of regular appeal rounds before the final round is triggered
        - **Final round lock terms:** Number of terms that a coherent guardian in a final round is disallowed to withdraw
        - **Appeal collateral factor:** ‱ multiple of dispute fees (guardians, draft, and settlements) required to appeal a preliminary ruling (1/10,000)
        - **Appeal confirmation collateral factor:** ‱ multiple of dispute fees (guardians, draft, and settlements) required to confirm an appeal (1/10,000)
        - **Min active balance:** Minimum amount of guardian tokens that can be activated
- **Authentication:** Open
- **Pre-flight checks:**
    - Ensure that the term duration does not last longer than a year
    - Ensure that the first Protocol term has not started yet
    - Ensure that the first-term start time is at least scheduled one Protocol term ahead in the future
    - Ensure that the first-term start time is scheduled earlier than 2 years in the future
    - Ensure that each dispute phase duration is not longer than 8670 terms
    - Ensure that the penalty permyriad is not above 10,000‱
    - Ensure that the final round reduction permyriad is not above 10,000‱
    - Ensure that the first round guardians number is greater than zero
    - Ensure that the number of max regular appeal rounds is between [1-10]
    - Ensure that the appeal step factor is greater than zero
    - Ensure that the appeal collateral factor is greater than zero
    - Ensure that the appeal confirmation collateral factor is greater than zero
    - Ensure that the minimum guardians active balance is greater than zero
- **State transitions:**
    - Save the Protocol term duration
    - Create a new term object for the first Protocol term
    - Create the initial Protocol configuration object
    - Create the governor object

#### Fallback
- **Actor:** Any external entity
- **Inputs:** Arbitrary
- **Authentication:** Open
- **Pre-flight checks:**
    - Ensure that the given signature has been registered as a custom function
- **State transitions:**
    - Forward call to the target registered for the custom function unless it is already implemented by the controller

#### Set config

- **Actor:** External entity in charge of maintaining the protocol configuration (config governor)
- **Inputs:**
    - **From term ID:** Identification number of the term in which the config will be effective at
    - **Settings:** Object containing
        - **Fee token:** Address of the token contract that is used to pay for the fees
        - **Guardian fee:** Amount of fee tokens paid per drafted guardian per dispute
        - **Draft fee:**  Amount of fee tokens per guardian to cover the drafting costs
        - **Settle fee:** Amount of fee tokens per guardian to cover round settlement costs
        - **Evidence terms:** Max submitting evidence period duration in Protocol terms
        - **Commit terms:** Duration of the commit phase in Protocol terms
        - **Reveal terms:** Duration of the reveal phase in Protocol terms
        - **Appeal terms:** Duration of the appeal phase in Protocol terms
        - **Appeal confirmation terms:** Duration of the appeal confirmation phase in Protocol terms
        - **Penalty permyriad:** ‱ of min active tokens balance to be locked for each drafted guardian (1/10,000)
        - **Final-round reduction:** ‱ of fee reduction for the last appeal round (1/10,000)
        - **First-round guardians number:** Number of guardians to be drafted for the first round of a dispute
        - **Appeal step factor:** Increasing factor for the number of guardians of each dispute round
        - **Max regular appeal rounds:** Number of regular appeal rounds before the final round is triggered
        - **Final round lock terms:** Number of terms that a coherent guardian in a final round is disallowed to withdraw
        - **Appeal collateral factor:** ‱ multiple of dispute fees (guardians, draft, and settlements) required to appeal a preliminary ruling (1/10,000)
        - **Appeal confirmation collateral factor:** ‱ multiple of dispute fees (guardians, draft, and settlements) required to confirm an appeal (1/10,000)
        - **Min active balance:** Minimum amount of guardian tokens that can be activated
- **Authentication:** Only config governor
- **Pre-flight checks:**
    - Ensure that the Protocol term is up-to-date. If not, perform a heartbeat before continuing the execution
    - Ensure that the config changes are being scheduled at least 2 terms in the future
    - Ensure that each dispute phase duration is not longer than 8670 terms
    - Ensure that the penalty permyriad is not above 10,000‱
    - Ensure that the final round reduction permyriad is not above 10,000‱
    - Ensure that the first round guardians number is greater than zero
    - Ensure that the number of max regular appeal rounds is between [1-10]
    - Ensure that the appeal step factor is greater than zero
    - Ensure that the appeal collateral factor is greater than zero
    - Ensure that the appeal confirmation collateral factor is greater than zero
    - Ensure that the minimum guardians active balance is greater than zero
- **State transitions:**
    - Update current Protocol term if needed
    - Create a new Protocol configuration object
    - Create a new future term object for the new configuration

#### Delay start time

- **Actor:** External entity in charge of maintaining the protocol configuration (config governor)
- **Inputs:**
    - **New first-term start time:** New timestamp in seconds when the Protocol will start
- **Authentication:** Allowed only to the config governor
- **Pre-flight checks:**
    - Ensure that the Protocol has not started yet
    - Ensure that the new proposed start time is in the future
- **State transitions:**
    - Update the protocol first term start time

#### Heartbeat

- **Actor:** Any entity incentivized to keep to Protocol term updated
- **Inputs:**
    - **Max allowed transitions:** Maximum number of transitions allowed, it can be set to zero to denote all the required transitions to update the Protocol to the current term
- **Authentication:** Open
- **Pre-flight checks:**
    - Ensure that the number of terms to be updated is greater than zero
- **State transitions:**
    - Update the Protocol term
    - Create a new term object for each transitioned new term

#### Ensure current term

- **Actor:** Any entity incentivized to keep to Protocol term updated
- **Inputs:** None
- **Authentication:** Open
- **Pre-flight checks:**
    - Ensure that the required number of transitions to update the Protocol term is not huge
- **State transitions:**
    - If necessary, update the Protocol term and create a new term object for each transitioned new term

#### Ensure current term randomness

- **Actor:** Any entity incentivized to compute the term randomness for the current term
- **Inputs:** None
- **Authentication:** Open
- **Pre-flight checks:**
    - Ensure a term object with that ID exists
- **State transitions:**
    - In case the term randomness has not been computed yet, set its randomness using the block hash of the following block when the term object was created

#### Set automatic withdrawals

- **Actor:** External entity holding funds in the protocol
- **Inputs:**
    - **Allowed:** Whether the automatic withdrawals for the sender are allowed or not
- **Authentication:** Open
- **Pre-flight checks:** None
- **State transitions:**
    - Update the automatic withdrawals config of the sender

#### Change funds governor

- **Actor:** External entity in charge of maintaining the protocol funds (funds governor)
- **Inputs:**
    - **New funds governor:** Address of the new funds governor to be set
- **Authentication:** Only funds governor
- **Pre-flight checks:**
    - Ensure that the new funds governor address is not zero
- **State transitions:**
    - Update the funds governor address

#### 4.2.10. Change config governor

- **Actor:** External entity in charge of maintaining the protocol configuration (config governor)
- **Inputs:**
    - **New config governor:** Address of the new config governor to be set
- **Authentication:** Allowed only to the config governor
- **Pre-flight checks:**
    - Ensure that the new config governor address is not zero
- **State transitions:**
    - Update the config governor address

#### 4.2.11. Change modules governor

- **Actor:** External entity in charge of maintaining the protocol modules (modules governor)
- **Inputs:**
    - **New modules governor:** Address of the new modules governor to be set
- **Authentication:** Allowed only to the modules governor
- **Pre-flight checks:**
    - Ensure that the new modules governor address is not zero
- **State transitions:**
    - Update the modules governor address

#### 4.2.12. Eject funds governor

- **Actor:** External entity in charge of maintaining the protocol funds (funds governor)
- **Inputs:** None
- **Authentication:** Only funds governor
- **Pre-flight checks:** None
- **State transitions:**
    - Unset the funds governor address

#### 4.2.13. Eject modules governor

- **Actor:** External entity in charge of maintaining the protocol modules (modules governor)
- **Inputs:** None
- **Authentication:** Only modules governor
- **Pre-flight checks:** None
- **State transitions:**
    - Unset the modules governor address

#### 4.2.14. Set module

- **Actor:** External entity in charge of maintaining the protocol modules (modules governor)
- **Inputs:**
    - **Module ID:** ID of the module to be set
    - **Address:** Address of the module to be set
- **Authentication:** Only modules governor
- **Pre-flight checks:**
    - Ensure that the module address is a contract
- **State transitions:**
    - Set the module address for the corresponding module ID

#### 4.2.15. Set modules

- **Actor:** External entity in charge of maintaining the protocol modules (modules governor)
- **Inputs:**
    - **New modules' IDs:** List of IDs of the new modules to be set
    - **New modules' addresses:** List of addresses of the new modules to be set
    - **New modules' links:** List of IDs of the modules to be linked in the new modules being set
    - **Current modules to be synced:** List of addresses of current modules to be re-linked to the new modules being set
- **Authentication:** Only modules governor
- **Pre-flight checks:**
    - Ensure both input lists have the same length
    - Ensure that the module addresses are contracts
    - Ensure that all the modules to be linked actually exist
- **State transitions:**
    - Save all the modules' addresses for their corresponding module ID
    - Link the implementations of the requested module IDs in the new modules set
    - Link the implementations of the new modules set in the requested current modules

#### 4.2.16. Sync module links

- **Actor:** External entity in charge of maintaining the protocol modules (modules governor)
- **Inputs:**
    - **Modules to be synced:** List of addresses of connected modules whose implementation links should be synced for the requested module ids
    - **IDs to be set:** List of IDs of the modules to be included in the sync
- **Authentication:** Only modules governor
- **Pre-flight checks:**
    - Ensure both input lists have at least one item
    - Ensure that all the modules to be linked actually exist
- **State transitions:**
    - Link the implementations of the requested module IDs in each of the requested modules

#### 4.2.17. Disable module

- **Actor:** External entity in charge of maintaining the protocol modules (modules governor)
- **Inputs:**
    - **Address:** Address of the module to be disabled
- **Authentication:** Only modules governor
- **Pre-flight checks:**
    - Ensure that the given module exists
    - Ensure that the given module is not disabled
- **State transitions:**
    - Mark the given module as disabled

#### 4.2.18. Enable module

- **Actor:** External entity in charge of maintaining the protocol modules (modules governor)
- **Inputs:**
    - **Address:** Address of the module to be enabled
- **Authentication:** Only modules governor
- **Pre-flight checks:**
    - Ensure that the given module exists
    - Ensure that the given module is not enabled
- **State transitions:**
    - Mark the given module as enabled

#### 4.2.19. Set custom function

- **Actor:** External entity in charge of maintaining the protocol modules (modules governor)
- **Inputs:**
    - **Signature:** Signature of the function to be customized
    - **Address:** Address of the target that will be forwarded with the function call
- **Authentication:** Only modules governor
- **Pre-flight checks:** None
- **State transitions:**
    - Set the target address for the given signature

#### 4.2.20. Grant

- **Actor:** External entity in charge of maintaining the protocol
- **Inputs:**
    - **Role:** ID of the role to be granted
    - **Address:** Address of the entity to grant the role to
- **Authentication:** Only config governor
- **Pre-flight checks:**
    - Ensure the role is not frozen
    - Ensure the entity does not have the role
- **State transitions:**
    - Add the role from the requested entity

#### 4.2.21. Revoke

- **Actor:** External entity in charge of maintaining the protocol
- **Inputs:**
    - **Role:** ID of the role to be revoked
    - **Address:** Address of the entity to revoke the role from
- **Authentication:** Only config governor
- **Pre-flight checks:**
    - Ensure the role is not frozen
    - Ensure the entity has the role
- **State transitions:**
    - Remove the role from the requested entity

#### 4.2.22. Freeze

- **Actor:** External entity in charge of maintaining the protocol
- **Inputs:**
    - **Role:** ID of the role to be frozen
- **Authentication:** Only config governor
- **Pre-flight checks:**
    - Ensure the given role is not frozen
- **State transitions:**
    - Mark the requested role as frozen

#### 4.2.23. Bulk

- **Actor:** External entity in charge of maintaining the protocol
- **Inputs:**
    - **Ops:** List of requested operations
    - **Roles:** List of roles for the requested operations
    - **Whos:** List of addresses for the requested operations
- **Authentication:** Only config governor
- **Pre-flight checks:**
    - Ensure the lists size match
- **State transitions:**
    - Execute all the requested ACL operations

### 4.3. Dispute Manager

The `DisputeManager` module is in charge of handling all the disputes-related behavior. This is where disputes are created and appealed.
It is also in charge of computing the final ruling for each dispute, and to settle the rewards and penalties of all the parties involved in the dispute.

#### 4.3.1. Constructor

- **Actor:** Deployer account
- **Inputs:**
    - **Controller:** Address of the `Controller` contract that centralizes all the modules being used
    - **Max guardians per draft batch:** Max number of guardians to be drafted in each batch
- **Authentication:** Open
- **Pre-flight checks:**
    - Ensure that the controller address is a contract
    - Ensure that the max number of guardians to be drafted per batch is greater than zero
- **State transitions:**
    - Save the controller address
    - Save the max number of guardians to be drafted per batch

#### 4.3.2. Create dispute

- **Actor:** Controller
- **Inputs:**
    - **Subject:** Arbitrable instance creating the dispute
    - **Possible rulings:** Number of possible results for a dispute
    - **Metadata:** Optional metadata that can be used to provide additional information on the dispute to be created
- **Authentication:** Only the controller is allowed to call this function
- **Pre-flight checks:**
    - Ensure that the Protocol term is up-to-date. Otherwise, perform a heartbeat before continuing the execution.
    - Ensure that the number of possible rulings is within some reasonable bounds (hardcoded as constants)
- **State transitions:**
    - Update current Protocol term if needed
    - Create a new dispute object
    - Create an adjudication round object setting the draft term at the end of the evidence submission period
    - Calculate dispute fees based on the current Protocol configuration
    - Set the number of guardians to be drafted based on the Protocol configuration at the term when dispute was created
    - Pull the required dispute fee amount from the sender to be deposited in the `Treasury` module, revert if the ERC20-transfer wasn't successful

#### 4.3.3. Close evidence period

- **Actor:** Controller
- **Inputs:**
    - **Dispute ID:** Dispute identification number
- **Authentication:** Only the controller is allowed to call this function
- **Pre-flight checks:**
    - Ensure that the Protocol term is up-to-date. Otherwise, perform a heartbeat before continuing the execution.
    - Ensure a dispute object with that ID exists
    - Ensure that the current term is at least after the term when the dispute was created
    - Ensure that the dispute evidence period is still open
- **State transitions:**
    - Update the dispute draft term ID of the first adjudication round to the current term

#### 4.3.4. Draft

- **Actor:** External entity incentivized by the draft fee they will earn by performing this execution. Alternatively, an altruistic entity to make sure the dispute is drafted
- **Inputs:**
    - **Dispute ID:** Dispute identification number
- **Authentication:** Open
- **Pre-flight checks:**
    - Ensure that the Protocol term is up-to-date. Otherwise, revert (cannot heartbeat and draft in the same block)
    - Ensure a dispute object with that ID exists
    - Ensure that the last round of the dispute hasn't finished drafting yet
    - Ensure that the draft term for the last round has been reached
    - Ensure that the randomness seed for the current term is either available (current block number within a certain range) or was saved by another draft
- **State transitions:**
    - Search up to the maximum batch size of guardians in the `GuardianRegistry` using the current term's randomness seed for entropy, which will lock a certain amount of ANT tokens to each of the drafted guardians based on the penalty permille of the Protocol. The maximum number of guardians to be drafted will depend on the maximum number allowed per batch set in the Protocol and the corresponding number of guardians for the dispute. Additionally, the `GuardiansRegistry` could return fewer guardians than the requested number. To have a better understanding of how the sortition works go to **section X**.
    - Update the dispute object with the resultant guardians from the draft. If all the guardians of the dispute have been drafted, transition the dispute to the adjudication phase.
    - Reward the caller with draft fees for each guardian drafted, using the configuration at the term when the dispute was created.

#### 4.3.5. Create appeal

- **Actor:** External entity not in favor of the ruling decided by the drafted guardians during the adjudication phase
- **Inputs:**
    - **Dispute ID:** Dispute identification number
    - **Round ID:** Adjudication round identification number
    - **Ruling:** Ruling number proposed by the appealer
- **Authentication:** Open. Implicitly, only accounts that have open an ERC20 allowance with an amount of at least the required appeal collateral to the `DisputeManager` module can call this function
- **Pre-flight checks:**
    - Ensure that the Protocol term is up-to-date. Otherwise, perform a heartbeat before continuing the execution.
    - Ensure a dispute object with that ID exists
    - Ensure an adjudication round object with that ID exists for the given dispute
    - Ensure that the adjudication round can be appealed
    - Ensure that the given ruling is different from the one already decided by the drafted guardians
    - Ensure that the given ruling is either refused or one of the possible rulings supported by the `DisputeManager` module
- **State transitions:**
    - Update current Protocol term if needed
    - Create a new appeal object tracking the address and proposed ruling of the appealer
    - Pull the required appeal collateral from the sender to be deposited in the `Treasury` module, calculating it based on the Protocol configuration when the dispute was created, revert if the ERC20-transfer wasn't successful

#### 4.3.6. Confirm appeal

- **Actor:** External entity not in favor of the ruling proposed by a previously submitted appeal
- **Inputs:**
    - **Dispute ID:** Dispute identification number
    - **Round ID:** Adjudication round identification number
    - **Ruling:** Ruling number proposed by the entity confirming the appeal
- **Authentication:** Open. Implicitly, only accounts that have open an ERC20 allowance with an amount of at least the required appeal confirmation collateral to the `DisputeManager` module can call this function
- **Pre-flight checks:**
    - Ensure that the Protocol term is up-to-date. Otherwise, perform a heartbeat before continuing the execution.
    - Ensure a dispute object with that ID exists
    - Ensure an adjudication round object with that ID exists for the given dispute
    - Ensure that the adjudication round was appeal and can still be confirmed
    - Ensure that the given ruling is different from the one proposed by the appealer
    - Ensure that the given ruling is either refused or one of the possible rulings supported by the `DisputeManager` module
- **State transitions:**
    - Update current Protocol term if needed
    - Create a new adjudication round object and set the draft term right after the end of the final adjudication phase of the current dispute round
    - If the final appeal round hasn't been reached yet:
        - Calculate the number of guardians to be drafted for the new round applying the appeal step factor to the number of guardians drafted for the previous round
        - Transition the dispute to the draft phase
    - If the final appeal round has been reached:
        - Calculate the number of guardians of the final round as the number of times the minimum ANT active balance is held in the `GuardiansRegistry` module
        - Transition the dispute to the adjudication phase
    - Update the current round appeal object tracking the address and proposed ruling of the account confirming the appeal
    - Calculate new round fees based on the Protocol configuration at the term when the dispute was created
    - Pull the required appeal confirmation collateral, which includes the new round fees, from the sender to be deposited in the `Treasury` module, revert if the ERC20-transfer wasn't successful

#### 4.3.7. Compute ruling

- **Actor:** External entity incentivized to execute the final ruling decided for a dispute. Alternatively, an altruistic entity to make sure the dispute is ruled.
- **Inputs:**
    - **Dispute ID:** Dispute identification number
- **Authentication:** Open
- **Pre-flight checks:**
    - Ensure a dispute object with that ID exists
    - Ensure that the dispute's last round adjudication phase has ended
- **State transitions:**
    - Update the final ruling of the dispute object based on the ruling decided by the guardians during the current round or the ruling proposed by the appealer of the previous round in case there was one but wasn't confirmed.

#### 4.3.8. Settle penalties

- **Actor:** External entity incentivized to slash the losing guardians. Alternatively, an altruistic entity to make sure the dispute is settled.
- **Inputs:**
    - **Dispute ID:** Dispute identification number
    - **Round ID:** Adjudication round identification number
    - **Max number of guardians to settle:** Maximum number of guardians to be settled during the call. It can be set to zero to denote all the guardians that were drafted for the adjudication round.
- **Authentication:** Open
- **Pre-flight checks:**
    - Ensure that the Protocol term is up-to-date. Otherwise, perform a heartbeat before continuing the execution.
    - Ensure a dispute object with that ID exists
    - Ensure an adjudication round object with that ID exists for the given dispute
    - Ensure that the given round is the first adjudication round of the dispute or that the previous round penalties were already settled
    - Ensure that the adjudication round penalties haven't been settled yet
- **State transitions:**
    - Update current Protocol term if needed
    - In case the final ruling of the dispute has not been computed yet, update the final ruling of the dispute object based on the ruling decided by the guardians during the current round or the ruling proposed by the appealer of the previous round in case there was one but wasn't confirmed.
    - Update the adjudication round object with the number of guardians that voted in favor of the final ruling
    - If the adjudication round being settled is not a final round:
        - Ask the `GuardiansRegistry` module to slash or unlock the locked ANT tokens from the drafted guardians based on whether they voted in favor of the dispute's final ruling or not.
        - Update the adjudication round object marking all the guardians whose penalties were settled
        - Deposit the corresponding settle fees based on the number of guardians settled to the caller in the `Treasury` module
    - If the adjudication round being settled is a final round:
        - Update the adjudication round object to mark that all guardians' penalties were settled
    - In case all the guardians' penalties have been settled, and there was not even one guardian voting in favor of the final ruling:
        - Ask the `GuardiansRegistry` module to burn all the ANT tokens that were collected during the adjudication round
        - Return the adjudication round fees to the dispute creator or the appeal parties depending on whether the adjudication round was triggered by the `Arbitrable` instance who created the dispute or due to a previous round that was appealed respectively.

#### 4.3.9. Settle reward

- **Actor:** External entity incentivized to reward the winning guardians. Alternatively, an altruistic entity to make sure the dispute is settled.
- **Inputs:**
    - **Dispute ID:** Dispute identification number
    - **Round ID:** Adjudication round identification number
    - **Guardian address:** Address of the guardian to settle their rewards
- **Authentication:** Open
- **Pre-flight checks:**
    - Ensure a dispute object with that ID exists
    - Ensure an adjudication round object with that ID exists for the given dispute
    - Ensure penalties have been settled
    - Ensure that the guardian has not been rewarded
    - Ensure that the guardian was drafted for the adjudication round
    - Ensure that the guardian has voted in favor of the final ruling of the dispute
- **State transitions:**
    - Update the adjudication round object marking that the guardian was rewarded
    - Assign to the guardian the corresponding portion of ANT tokens slashed from the losing guardians
    - Deposit the corresponding portion of guardian fees into the `Treasury` module to the guardian

#### 4.3.10. Settle appeal deposit

- **Actor:** External entity incentivized to settle the appeal parties. Alternatively, an altruistic entity to make sure the dispute is settled.
- **Inputs:**
    - **Dispute ID:** Dispute identification number
    - **Round ID:** Adjudication round identification number
- **Authentication:** Open
- **Pre-flight checks:**
    - Ensure a dispute object with that ID exists
    - Ensure an adjudication round object with that ID exists for the given dispute
    - Ensure penalties have been settled
    - Ensure that the adjudication round was appealed
    - Ensure that the adjudication round's appeal has not been settled yet
- **State transitions:**
    - Mark the adjudication round's appeal as settled
    - Deposit the corresponding portions of the appeal deposits into the `Treasury` module to each party

#### 4.3.11. Ensure can commit

- **Actor:** Any entity incentivized to check if it is possible to commit votes for a certain dispute adjudication round
- **Inputs:**
    - **Vote ID:** Vote identification number
- **Authentication:** Open
- **Pre-flight checks:**
    - Ensure that the Protocol term is up-to-date. Otherwise, perform a heartbeat before continuing the execution.
    - Ensure a dispute and adjudication round exists with that vote ID
    - Ensure votes can still be committed for the adjudication round
- **State transitions:**
    - Update current Protocol term if needed


#### 4.3.12. Ensure voter can commit

- **Actor:** Any entity incentivized to check if it is possible to commit votes for a certain dispute adjudication round
- **Inputs:**
    - **Vote ID:** Vote identification number
- **Authentication:** Only active `Voting` modules
- **Pre-flight checks:**
    - Ensure that the Protocol term is up-to-date. Otherwise, perform a heartbeat before continuing the execution.
    - Ensure a dispute and adjudication round exists with that vote ID
    - Ensure votes can still be committed for the adjudication round
    - Ensure that the voter was drafted to vote for the adjudication round
- **State transitions:**
    - Update current Protocol term if needed
    - Update the guardian's weight for the adjudication round if its a final round

#### 4.3.13. Ensure voter can reveal

- **Actor:** Any entity incentivized to check if it is possible to reveal votes for a certain dispute adjudication round
- **Inputs:**
    - **Vote ID:** Vote identification number
- **Authentication:**
- **Pre-flight checks:**
    - Ensure that the Protocol term is up-to-date. Otherwise, perform a heartbeat before continuing the execution.
    - Ensure a dispute and adjudication round exists with that vote ID
    - Ensure votes can still be revealed for the adjudication round
- **State transitions:**
    - Update current Protocol term if needed

#### 4.3.14. Set max guardians per draft batch

- **Actor:** External entity in charge of maintaining the protocol configuration (config governor)
- **Inputs:**
    - **New max guardians per draft batch:** New max number of guardians to be drafted in each batch
- **Authentication:** Only config governor
- **Pre-flight checks:**
    - Ensure that the max number of guardians to be drafted per batch is greater than zero
- **State transitions:**
    - Save the max number of guardians to be drafted per batch

#### 4.3.15. Recover funds

- **Actor:** External entity in charge of maintaining the protocol funds (funds governor)
- **Inputs:**
    - **Token:** Address of the ERC20-compatible token or ETH to be recovered from the `DisputeManager` module
    - **Recipient:** Address that will receive the funds of the `DisputeManager` module
- **Authentication:** Only funds governor
- **Pre-flight checks:**
    - Ensure that the balance of the `DisputeManager` module is greater than zero
- **State transitions:**
    - Transfer the whole balance of the `DisputeManager` module to the recipient address, revert if the ERC20-transfer wasn't successful

### 4.4. Guardians Registry

The `GuardiansRegistry` module is in charge of handling the guardians activity and mainly the different states of their staked balances.
This module is in the one handling all the staking/unstaking logic for the guardians, all the ANT staked into the Protocol is held by the registry.

#### 4.4.1. Constructor

- **Actor:** Deployer account
- **Inputs:**
    - **Controller:** Address of the `Controller` contract that centralizes all the modules being used
    - **Guardian token:** Address of the ERC20 token to be used as guardian token for the registry
    - **Total active balance limit:** Maximum amount of total active balance that can be held in the registry
- **Authentication:** Open
- **Pre-flight checks:**
    - Ensure that the controller address is a contract
    - Ensure that the guardian token address is a contract
    - Ensure that the total active balance limit is greater than zero
- **State transitions:**
    - Save the controller address
    - Save the guardian token address
    - Save the total active balance limit

#### 4.4.2. Stake

- **Actor:** Guardian or an external entity incentivized in a guardian of the Protocol
- **Inputs:**
    - **Guardian:** Address of the guardian staking the tokens for
    - **Amount:** Amount of tokens to be staked
- **Authentication:** Open. Implicitly, only senders that have open an ERC20 allowance with the requested amount of tokens to stake.
- **Pre-flight checks:**
    - Ensure that the given amount is greater than zero
- **State transitions:**
    - Update the available balance of the guardian
    - Pull the corresponding amount of guardian tokens from the sender to the `GuardiansRegistry` module, revert if the ERC20-transfer wasn't successful

#### 4.4.3. Unstake

- **Actor:** Guardian or an authorized role holder
- **Inputs:**
    - **Guardian:** Address of the guardian unstaking the tokens from
    - **Amount:** Amount of tokens to be unstaked
- **Authentication:** The guardian or an authorized role holder. Only for guardians that have some available balance in the registry.
- **Pre-flight checks:**
    - Ensure that the requested amount is greater than zero
    - Ensure that there is enough available balance for the requested amount
- **State transitions:**
    - Update the available balance of the guardian
    - Process previous deactivation requests if there is any, increase the guardian's available balance
    - Transfer the requested amount of guardian tokens from the `GuardiansRegistry` module to the guardian, revert if the ERC20-transfer wasn't successful

#### 4.4.4. Activate

- **Actor:** Guardian or an authorized role holder
- **Inputs:**
    - **Guardian:** Address of the guardian activating the tokens for
    - **Amount:** Amount of guardian tokens to be activated for the next term
- **Authentication:** The guardian or an authorized role holder. Only for guardians with some available balance.
- **Pre-flight checks:**
    - Ensure that the Protocol term is up-to-date. Otherwise, perform a heartbeat before continuing the execution.
    - Ensure that the requested amount is greater than zero
    - Ensure that the guardian's available balance is enough for the requested amount
    - Ensure that the new active balance is greater than the minimum active balance for the Protocol
    - Ensure that the total active balance held in the registry does not reach the limit
- **State transitions:**
    - Update current Protocol term if needed
    - Process previous deactivation requests if there is any, increase the guardian's available balance
    - Update the guardian's active balance for the next term
    - Decrease the guardian's available balance

#### 4.4.5. Deactivate

- **Actor:** Guardian or an authorized role holder
- **Inputs:**
    - **Guardian:** Address of the guardian deactivating the tokens for
    - **Amount:** Amount of guardian tokens to be deactivated for the next term
- **Authentication:** The guardian or an authorized role holder. Only for guardians with some activated balance.
- **Pre-flight checks:**
    - Ensure that the Protocol term is up-to-date. Otherwise, perform a heartbeat before continuing the execution.
    - Ensure that the unlocked active balance of the guardians is enough for the requested amount
    - Ensure that the remaining active balance is either zero or greater than the minimum active balance for the Protocol
- **State transitions:**
    - Update current Protocol term if needed
    - Process previous deactivation requests if there is any, increase the guardian's available balance
    - Create a new deactivation request object for the next term

#### 4.4.6. Stake and activate

- **Actor:** Guardian or an authorized role holder
- **Inputs:**
    - **Guardian:** Address of the guardian to stake and activate an amount of tokens to
    - **Amount:** Amount of tokens to be staked
- **Authentication:** The guardian or an authorized role holder. Only if the sender has open an ERC20 allowance with the requested amount of tokens to stake can call this function.
- **Pre-flight checks:**
    - Validate that the sender is the guardian himself or an authorized role holder
    - Ensure that the given amount is greater than zero
- **State transitions:**
    - Update the available balance of the guardian
    - Activate the staked amount if requested. This includes processing pending deactivation requests.
    - Pull the corresponding amount of guardian tokens from the sender to the `GuardiansRegistry` module, revert if the ERC20-transfer wasn't successful

#### 4.4.7. Lock activation

- **Actor:** Guardian or an authorized role holder
- **Inputs:**
    - **Guardian:** Address of the guardian lock the activation for
    - **Lock manager**: Address of the lock manager that will control the lock
    - **Amount**: Amount of active tokens to be locked
- **Authentication:** Only the guardian or an authorized role holder
- **Pre-flight checks:**
    - Ensure that the given lock manager is authorized
- **State transitions:**
    - Increase the total amount locked for the guardian
    - Increase the amount locked for the guardian by the given lock manager

#### 4.4.8. Unlock activation

- **Actor:** External entity incentivized to unlock the activation of a guardian of the Protocol
- **Inputs:**
    - **Guardian:** Address of the guardian unlocking the active balance of
    - **Lock manager:** Address of the lock manager controlling the lock
    - **Amount:** Amount of active tokens to be unlocked
    - **Request deactivation:** Whether the unlocked amount must be requested for deactivation immediately
- **Authentication:** Only the guardian or an authorized role holder. Only if the lock manager allows to unlock the requested amount.
- **Pre-flight checks:**
    - Ensure that the requested amount can be unlocked
    - Ensure that the given amount is greater than zero
    - Ensure that the given lock manager has locked some amount
    - If a deactivation is requested, check the sender is the guardian or an authorized role holder
- **State transitions:**
    - Decrease the total amount locked for the guardian
    - Decrease the amount locked for the guardian by the given lock manager
    - Schedule a deactivation if requested

#### 4.4.9. Process deactivation request

- **Actor:** External entity incentivized to update guardians available balances
- **Inputs:**
    - **Guardian:** Address of the guardian to process the deactivation request of
- **Authentication:** Open
- **Pre-flight checks:**
    - Ensure that the Protocol term is up-to-date. Otherwise, perform a heartbeat before continuing the execution.
    - Ensure there is an existing deactivation request for the guardian
    - Ensure that the existing deactivation request can be processed at the current term
- **State transitions:**
    - Increase the available balance of the guardian
    - Reset the deactivation request of the guardian

#### 4.4.10. Assign tokens

- **Actor:** `DisputeManager` module
- **Inputs:**
    - **Guardian:** Address of the guardian to add an amount of tokens to
    - **Amount:** Amount of tokens to be added to the available balance of a guardian
- **Authentication:** Only active `DisputeManager` modules
- **Pre-flight checks:** None
- **State transitions:**
    - Increase the guardian's available balance

#### 4.4.11. Burn tokens

- **Actor:** `DisputeManager` module
- **Inputs:**
    - **Amount:** Amount of tokens to be burned
- **Authentication:** Only active `DisputeManager` modules
- **Pre-flight checks:** None
- **State transitions:**
    - Increase the burn address's available balance

#### 4.4.12. Draft

- **Actor:** `DisputeManager` module
- **Inputs:**
    - **Draft params:** Object containing:
        - **Term randomness:** Randomness to compute the seed for the draft
        - **Dispute ID:** Identification number of the dispute to draft guardians for
        - **Term ID:** Identification number of the current term when the draft is being computed
        - **Selected guardians:** Number of guardians already selected for the draft
        - **Batch requested guardians:** Number of guardians to be selected in the given batch of the draft
        - **Draft requested guardians:** Total number of guardians requested to be drafted
        - **Draft locking permyriad:** ‱ of the minimum active balance to be locked for the draft (1/10,000)
- **Authentication:** Only active `DisputeManager` modules
- **Pre-flight checks:**
    - Ensure that the requested number of guardians to be drafted is greater than zero
    - Ensure each drafted guardian has enough active balance to be locked for the draft
    - Ensure that a limit number of drafting iterations will be computed
- **State transitions:**
    - Update the locked active balance of each drafted guardian
    - Decrease previous deactivation requests if there is any and needed to draft the guardian

#### 4.4.13. Slash or unlock

- **Actor:** `DisputeManager` module
- **Inputs:**
    - **Term ID:** Current term identification number
    - **Guardians:** List of guardian addresses to be slashed
    - **Locked amounts:** List of amounts locked for each corresponding guardian that will be either slashed or returned
    - **Rewarded guardians:** List of booleans to tell whether a guardian's active balance has to be slashed or not
- **Authentication:** Only active `DisputeManager` modules
- **Pre-flight checks:**
    - Ensure that both lists lengths match
    - Ensure that each guardian has enough locked balance to be unlocked
- **State transitions:**
    - Decrease the unlocked balance of each guardian based on their corresponding given amounts
    - In case of a guardian being slashed, decrease their active balance for the next term

#### 4.4.14. Collect tokens

- **Actor:** `DisputeManager` module
- **Inputs:**
    - **Guardian:** Address of the guardian to collect the tokens from
    - **Amount:** Amount of tokens to be collected from the given guardian and for the requested term id
    - **Term ID:** Current term identification number
- **Authentication:** Only active `DisputeManager` modules
- **Pre-flight checks:**
    - Ensure the guardian has enough active balance based on the requested amount
- **State transitions:**
    - Decrease the active balance of the guardian for the next term
    - Decrease previous deactivation requests if there is any and its necessary to collect the requested amount of tokens from a guardian

#### 4.4.15. Lock withdrawals

- **Actor:** `DisputeManager` module
- **Inputs:**
    - **Guardian:** Address of the guardian to locked the withdrawals of
    - **Term ID:** Term identification number until which the guardian's withdrawals will be locked
- **Authentication:** Only active `DisputeManager` modules
- **Pre-flight checks:** None
- **State transitions:**
    - Update the guardian's state with the term ID until which their withdrawals will be locked

#### 4.4.16. Set total active balance limit

- **Actor:** External entity in charge of maintaining the protocol configuration (config governor)
- **Inputs:**
    - **New total active balance limit:** New limit of total active balance of guardian tokens
- **Authentication:** Only config governor
- **Pre-flight checks:**
    - Ensure that the total active balance limit is greater than zero
- **State transitions:**
    - Update the total active balance limit

#### 4.4.17. Recover funds

- **Actor:** External entity in charge of maintaining the protocol funds (funds governor)
- **Inputs:**
    - **Token:** Address of the ERC20-compatible token or ETH to be recovered from the `GuardiansRegistry` module
    - **Recipient:** Address that will receive the funds of the `GuardiansRegistry` module
- **Authentication:** Only funds governor
- **Pre-flight checks:**
    - Ensure that the balance of the `GuardiansRegistry` module is greater than zero
- **State transitions:**
    - Transfer the whole balance of the `GuardiansRegistry` module to the recipient address, revert if the ERC20-transfer wasn't successful

### 4.5. Voting

The `Voting` module is in charge of handling all the votes submitted by the drafted guardians and computing the tallies to ensure the final ruling of a dispute once finished.
In particular, the first version of the protocol uses a commit-reveal mechanism. Therefore, the `Voting` module allows guardians to commit and reveal their votes, and leaked other guardians votes.

#### 4.5.1. Constructor

- **Actor:** Deployer account
- **Inputs:**
    - **Controller:** Address of the `Controller` contract that centralizes all the modules being used
- **Authentication:** Open
- **Pre-flight checks:**
    - Ensure that the controller address is a contract
- **State transitions:**
    - Save the controller address

#### 4.5.2. Delegate

- **Actor:** Any guardian that could potentially be drafted for an adjudication round or an authorized role holder
- **Inputs:**
    - **Voter:** Address of the voter setting their delegate
    - **Delegate:** Address of the delegate to be set
- **Authentication:** Open. Implicitly called by only potential guardians or an authorized role holder.
- **Pre-flight checks:** None
- **State transitions:**
    - Set the voter's delegate

#### 4.5.3. Create

- **Actor:** `DisputeManager` module
- **Inputs:**
    - **Vote ID:** Vote identification number
- **Authentication:** Only the current `DisputeManager` module
- **Pre-flight checks:**
    - Ensure there is no other existing vote for the given vote ID
- **State transitions:**
    - Create a new vote object

#### 4.5.3. Commit

- **Actor:** Guardian drafted for an adjudication round or an authorized role holder
- **Inputs:**
    - **Vote ID:** Vote identification number
    - **Voter:** Address of the voter committing the vote for
    - **Commitment:** Hashed outcome to be stored for future reveal
- **Authentication:** Only the voter or an authorized role holder. Implicitly, only guardians that were drafted for the corresponding adjudication round can call this function.
- **Pre-flight checks:**
    - Ensure a vote object with that ID exists
    - Ensure that the sender was drafted for the corresponding dispute's adjudication round
    - Ensure that the sender has not committed a vote before
    - Ensure that votes can still be committed for the adjudication round
- **State transitions:**
    - Create a cast vote object for the sender

#### 4.5.4. Leak

- **Actor:** External entity incentivized to slash a guardian
- **Inputs:**
    - **Vote ID:** Vote identification number
    - **Voter:** Address of the voter to leak a vote of
    - **Outcome:** Outcome leaked for the voter
    - **Salt:** Salt to decrypt and validate the committed vote of the voter
- **Authentication:** Open
- **Pre-flight checks:**
    - Ensure the voter commitment can be decrypted with the provided outcome and salt values
    - Ensure that votes can still be committed for the adjudication round
- **State transitions:**
    - Update the voter's cast vote object marking it as leaked

#### 4.5.5. Reveal

- **Actor:** Guardian drafted for an adjudication round
- **Inputs:**
    - **Vote ID:** Vote identification number
    - **Voter:** Address of the voter revealing a vote for
    - **Outcome:** Outcome leaked for the voter
    - **Salt:** Salt to decrypt and validate the committed vote of the voter
- **Authentication:** Open
- **Pre-flight checks:**
    - Ensure the voter commitment can be decrypted with the provided outcome and salt values
    - Ensure the resultant outcome is valid
    - Ensure that votes can still be revealed for the adjudication round
- **State transitions:**
    - Update the voter's cast vote object saving the corresponding outcome
    - Update the vote object tally

## 4.6. PaymentsBook

The `PaymentsBook` module is in charge of collecting any extra payments paid by users to use Aragon Protocol.
This module is simply in charge of collecting any type of payment and distributing it to the corresponding parties: guardians and the governor.
Aragon Protocol does not explicitly require users to provide these extra payments on-chain. The idea is that any custom mechanism can be built on top and then verified by guardians handling arising disputes.

#### 4.6.1. Constructor

- **Actor:** Deployer account
- **Inputs:**
    - **Controller:** Address of the `Controller` contract that centralizes all the modules being used
    - **Period duration:** Duration of the payment period in Protocol terms
    - **Governor share permyriad:** Initial ‱ of the collected payments that will be saved for the governor (1/10,000)
- **Authentication:** Open
- **Pre-flight checks:**
    - Ensure that the controller address is a contract
    - Ensure that the period duration is greater than zero
    - Ensure that the new governor share permyriad is not above 10,000‱
- **State transitions:**
    - Save the controller address
    - Save the period duration
    - Save the governor share permyriad

#### 4.6.2. Pay

- **Actor:** Users of the Protocol
- **Inputs:**
    - **Token:** Address of the token being used for the payment
    - **Amount:** Amount of tokens being paid
    - **Payer:** Address assigning the payment to
    - **Data:** Optional data to be logged
- **Authentication:** Open
- **Pre-flight checks:**
    - Ensure that the payment amount is greater than zero
- **State transitions:**
    - Update the total amount collected for guardians during the current period
    - Update the total amount collected for the governor during the current period
    - Pull the corresponding amount of tokens from the sender to be deposited in the `PaymentsBook` module, revert if the EC20-transfer wasn't successful or if the ETH received does not match the requested one

#### 4.6.3. Claim guardian share

- **Actor:** Guardians of the Protocol
- **Inputs:**
    - **Period ID:** Period identification number
    - **Guardian:** Address of the guardian claiming the shares for
    - **Tokens:** List of addresses of the tokens being claimed
- **Authentication:** Open. Implicitly, only guardians that have certain amount of ANT tokens activated during the requested period can call this function or an authorized role holder
- **Pre-flight checks:**
    - Ensure that the requested period has already ended
    - Ensure that the sender has not already claimed their share for the requested token and period
    - Ensure that the sender's share is greater than zero for the requested token and period
- **State transitions:**
    - Compute period balance checkpoint if it wasn't computed yet
    - Mark the sender's share as claimed for the requested period and token
    - Transfer the corresponding tokens to the sender, revert if the transfer wasn't successful

#### 4.6.4. Claim governor share

- **Actor:** External entity in charge of maintaining the protocol configuration (config governor)
- **Inputs:**
    - **Period ID:** Period identification number
    - **Tokens:** List of addresses of the tokens being claimed
- **Authentication:** Check the given period is a past period
- **Pre-flight checks:**
    - Ensure that the requested period has already ended
    - Ensure that the governor has not claimed their share for the requested token and period
    - Ensure that the governor's share is greater than zero for the requested token and period
- **State transitions:**
    - Mark the governor's share as claimed for the requested period and token
    - Transfer the corresponding tokens to the config governor address, revert if the transfer wasn't successful

#### 4.6.5. Ensure period balance details

- **Actor:** External entity incentivized in updating the parameters to determine the guardian's share for each period
- **Inputs:**
    - **Period ID:** Period identification number
- **Authentication:** Open
- **Pre-flight checks:**
    - Ensure that the last term included in the requested period has already started
- **State transitions:**
    - Pick a random term checkpoint included in the requested period using the next period's start term randomness, and save the total ANT active balance in the `GuardiansRegistry` at that term for the requested period

#### 4.6.12. Set governor share permyriad

- **Actor:** External entity in charge of maintaining the protocol configuration (config governor)
- **Inputs:**
    - **New governor share permyriad:** New ‱ of the collected payments that will be saved for the governor (1/10,000)
- **Authentication:** Only config governor
- **Pre-flight checks:**
    - Ensure that the new governor share permyriad is not above 10,000‱
- **State transitions:**
    - Update the governor share permyriad

#### 4.6.13. Recover funds

- **Actor:** External entity in charge of maintaining the protocol funds (funds governor)
- **Inputs:**
    - **Token:** Address of the ERC20-compatible token or ETH to be recovered from the `PaymentsBook` module
    - **Recipient:** Address that will receive the funds of the `PaymentsBook` module
- **Authentication:** Only funds governor
- **Pre-flight checks:**
    - Ensure that the balance of the `PaymentsBook` module is greater than zero
- **State transitions:**
    - Transfer the whole balance of the `PaymentsBook` module to the recipient address, revert if the transfer wasn't successful

### 4.7. Treasury

The `Treasury` module is in charge of handling the token assets related to the disputes process.
The staked ANT of the guardians and the payments fees of the users are the only assets excluded from the `Treasury`; those are handled in the `GuardianRegistry` and `PaymentBook`, respectively.
Except from those, the `Treasury` stores the rest of the fees, deposits, and collaterals required to back the different adjudication rounds of a dispute.

#### 4.7.1. Constructor

- **Actor:** Deployer account
- **Inputs:**
    - **Controller:** Address of the `Controller` contract that centralizes all the modules being used
- **Authentication:** Open
- **Pre-flight checks:**
    - Ensure that the controller address is a contract
- **State transitions:**
    - Save the controller address

#### 4.7.2. Assign

- **Actor:** `DisputeManager` module
- **Inputs:**
    - **Token:** Address of the ERC20-compatible token to be withdrawn
    - **Recipient:** Address that will receive the funds being withdrawn
    - **Amount:** Amount of tokens to be transferred to the recipient
- **Authentication:** Only active `DisputeManager` modules
- **Pre-flight checks:**
    - Ensure that the requested amount is greater than zero
- **State transitions:**
    - Increase the token balance of the recipient based on the requested amount

#### 4.7.3. Withdraw

- **Actor:** External entity owning a certain amount of tokens of the `Treasury` module or an authorized role holder.
- **Inputs:**
    - **Token:** Address of the ERC20-compatible token to be withdrawn
    - **From:** Address withdrawing the tokens from
    - **Recipient:** Address that will receive the funds being withdrawn
    - **Amount:** Amount of tokens to be transferred to the recipient
    - **Authorization:** Optional authorization granted by the voter in case of a third party sender
- **Authentication:** Open. Implicitly, only addresses that have some balance assigned in the `Treasury` module
- **Pre-flight checks:**
    - Validate signature if given
    - Ensure that the token balance of the caller is greater than zero
    - Ensure that the token balance of the caller is greater than or equal to the requested amount
- **State transitions:**
    - Update next nonce of the voter if a signature was given
    - Reduce the token balance of the caller based on the requested amount
    - Transfer the requested token amount to the recipient address, revert if the ERC20-transfer wasn't successful

#### 4.7.5. Recover funds

- **Actor:** External entity in charge of maintaining the protocol funds (funds governor)
- **Inputs:**
    - **Token:** Address of the ERC20-compatible token or ETH to be recovered from the `Treasury` module
    - **Recipient:** Address that will receive the funds of the `Treasury` module
- **Authentication:** Only funds governor
- **Pre-flight checks:**
    - Ensure that the balance of the `Treasury` module is greater than zero
- **State transitions:**
    - Transfer the whole balance of the `Treasury` module to the recipient address, revert if the ERC20-transfer wasn't successful

## 5. Data structures

The following sections aim to describe the different data structures used by each of the modules described in [section 4](../4-entry-points).

### 5.1. AragonProtocol

`AragonProtocol` does not rely on any custom data structure, but it relies on the data structures defined by the `Controller`. These can be explored in the [next section](./2-controller.md).

### 5.2. Controller

The following objects are the data-structures used by the `Controller`:

#### 5.2.1. Governor

The governor object includes the following fields:

- **Funds:** Address allowed to recover funds from the recoverable modules
- **Config:** Address allowed to change the different configurations of the whole system
- **Modules:** Address allowed to plug/unplug modules from the system

#### 5.2.2. Module

The module object includes the following fields:

- **ID:** ID associated to a module
- **Disabled:** Whether the module is disabled

#### 5.2.3. Config

The config object includes the following fields:

- **Fees config:** Fees config object
- **Disputes config:** Disputes config object
- **Min active balance:** Minimum amount of tokens guardians have to activate to participate in the Protocol

#### 5.2.4. Fees config

The fees config object includes the following fields:

- **Token:** ERC20 token to be used for the fees of the Protocol
- **Final round reduction:** Permyriad of fees reduction applied for final appeal round (1/10,000)
- **Guardian fee:** Amount of tokens paid to draft a guardian to adjudicate a dispute
- **Draft fee:** Amount of tokens paid per round to cover the costs of drafting guardians
- **Settle fee:** Amount of tokens paid per round to cover the costs of slashing guardians

#### 5.2.5. Disputes config

The disputes config object includes the following fields:

- **Evidence terms:** Max submitting evidence period duration in Protocol terms
- **Commit terms:** Committing period duration in terms
- **Reveal terms:** Revealing period duration in terms
- **Appeal terms:** Appealing period duration in terms
- **Appeal confirmation terms:** Confirmation appeal period duration in terms
- **Penalty permyriad:** ‱ of min active tokens balance to be locked for each drafted guardian (1/10,000)
- **First round guardians number:** Number of guardians drafted on first round
- **Appeal step factor:** Factor in which the guardians number is increased on each appeal
- **Final round lock terms:** Period a coherent guardian in the final round will remain locked
- **Max regular appeal rounds:** Before the final appeal
- **Appeal collateral factor:** Permyriad multiple of dispute fees (guardians, draft, and settlements) required to appeal a preliminary ruling (1/10,000)
- **Appeal confirmation collateral factor:** Permyriad multiple of dispute fees (guardians, draft, and settlements) required to confirm appeal (1/10,000)

#### 5.2.6. Term

The term object includes the following fields:

- **Start time:** Timestamp when the term started
- **Randomness block number:** Block number for entropy
- **Randomness:** Entropy from randomness block number's hash

### 5.3. Dispute Manager

The following objects are the data-structures used by the `DisputeManager`:

#### 5.3.1. Dispute

The dispute object includes the following fields:

- **Subject:** Arbitrable instance associated to a dispute
- **Possible rulings:** Number of possible rulings guardians can vote for each dispute
- **Creation term ID:** Identification number when the dispute was created
- **Final ruling:** Winning ruling of a dispute
- **Dispute state:** State of a dispute: pre-draft, adjudicating, or ruled
- **Adjudication rounds:** List of adjudication rounds for each dispute

#### 5.3.2. Adjudication round

The adjudication round object includes the following fields:

- **Draft term ID:** Term from which the guardians of a round can be drafted
- **Guardians number:** Number of guardians drafted for a round
- **Settled penalties:** Whether or not penalties have been settled for a round
- **Guardian fees:** Total amount of fees to be distributed between the winning guardians of a round
- **Guardians:** List of guardians drafted for a round
- **Guardians states:** List of states for each drafted guardian indexed by address
- **Delayed terms:** Number of terms a round was delayed based on its requested draft term id
- **Selected guardians:** Number of guardians selected for a round, to allow drafts to be batched
- **Coherent guardians:** Number of drafted guardians that voted in favor of the dispute final ruling
- **Settled guardians:** Number of guardians whose rewards were already settled
- **Collected tokens:** Total amount of tokens collected from losing guardians
- **Appeal:** Appeal-related information of a round

#### 5.3.3. Guardian state

The guardian state object includes the following fields:

- **Weight:** Weight computed for a guardian on a round
- **Rewarded:** Whether or not a drafted guardian was rewarded

#### 5.3.4. Appeal

The appeal object includes the following fields:

- **Maker:** Address of the appealer
- **Appealed ruling:** Ruling appealing in favor of
- **Taker:** Address of the one confirming an appeal
- **Opposed ruling:** Ruling opposed to an appeal
- **Settled:** Whether or not an appeal has been settled

### 5.3. Guardians Registry

The following objects are the data-structures used by the `GuardiansRegistry`:

#### 5.3.1. Guardian

The guardian object includes the following fields:

- **ID:** Identification number of each guardian
- **Locked balance:** Maximum amount of tokens that can be slashed based on the guardian's drafts
- **Active balance:** Tokens activated for the Protocol that can be locked in case the guardian is drafted
- **Available balance:** Available tokens that can be withdrawn at any time
- **Withdrawals lock term ID:** Term identification number until which guardian's withdrawals will be locked
- **Deactivation request:** Pending deactivation request of a guardian

#### 5.3.2. Deactivation request

The deactivation request object includes the following fields:

- **Amount:** Amount requested for deactivation
- **Available termId:** ID of the term when guardians can withdraw their requested deactivation tokens

#### 5.3.2. Activation locks

The activation locks object includes the following fields:

- **Total:** Total amount of active balance locked for a guardian
- **Available termId:** List of locked amounts indexed by lock manager

### 5.4. Voting

The following objects are the data-structures used by the `Voting`:

#### 5.4.1. Vote

The vote object includes the following fields:

- **Winning outcome:** Outcome winner of a vote instance
- **Max allowed outcome:** Highest outcome allowed for the vote instance
- **Cast votes:** List of cast votes indexed by voters addresses
- **Outcomes tally:** Tally for each of the possible outcomes

#### 5.4.2. Cast vote

The cast vote object includes the following fields:

- **Commitment:** Hash of the outcome casted by the voter
- **Outcome:** Outcome submitted by the voter

### 5.4. Voting

The following objects are the data-structures used by the `Voting`:

#### 5.4.1. Vote

The vote object includes the following fields:

- **Winning outcome:** Outcome winner of a vote instance
- **Max allowed outcome:** Highest outcome allowed for the vote instance
- **Cast votes:** List of cast votes indexed by voters addresses
- **Outcomes tally:** Tally for each of the possible outcomes

#### 5.4.2. Cast vote

The cast vote object includes the following fields:

- **Commitment:** Hash of the outcome casted by the voter
- **Outcome:** Outcome submitted by the voter

### 5.5. PaymentsBook

The following objects are the data-structures used by the `PaymentsBook`:

#### 5.5.1. Period

The period object includes the following fields:

- **Balance checkpoint:** Term identification number of a period used to fetch the total active balance of the guardians registry
- **Total active balance:** Total amount of guardian tokens active in the Protocol at the corresponding period checkpoint
- **Guardians shares:** List of total amount collected for the guardians during a period indexed by token
- **Governor shares:** List of total amount collected for the governor during a period indexed by token
- **Claimed guardians:** List of guardians that have claimed their share for a period, indexed by guardian address and token

### 5.7. Treasury

The `Treasury` module does not rely on any custom data structure.

## 6. External interface

The following sections aim to complement [section 4](../4-entry-points)'s description of each module's external entry points with their view-only access points and emitted events.

### 6.1. AragonProtocol

#### 6.1.1 Events

No custom events are implemented by `AragonProtocol`.

#### 6.1.2. Getters

The following functions are state getters provided by `AragonProtocol`:

##### 6.1.2.1. Dispute fees

- **Inputs:** None
- **Pre-flight checks:** None
- **Outputs:**
    **Recipient:** Address where the corresponding dispute fees must be transferred to
    **Fee token:** ERC20 token used for the fees
    **Fee amount:** Total amount of fees that must be allowed to the recipient

##### 6.1.2.2. Payments recipient

- **Inputs:** None
- **Pre-flight checks:** None
- **Outputs:**
    **Recipient:** Address where the payment fees must be transferred to

### 6.2. Controller

#### 6.2.1 Events

The following events are emitted by the `Controller`:

##### 6.2.1.1. Config changed

- **Name:** `NewConfig`
- **Args:**
    - **From term ID:** Identification number of the Protocol term when the config change will happen
    - **Protocol config ID:** Identification number of the Protocol config to be changed

##### 6.2.1.2. Start time delayed

- **Name:** `StartTimeDelayed`
- **Args:**
    - **Previous first term start time:** Previous timestamp in seconds when the Protocol will start
    - **Current first-term start time:** New timestamp in seconds when the Protocol will start

##### 6.2.1.3. Heartbeat

- **Name:** `Heartbeat`
- **Args:**
    - **Previous term ID:** Identification number of the Protocol term before the transition
    - **Current term ID:** Identification number of the Protocol term after the transition

##### 6.2.1.4. Automatic withdrawals changed

- **Name:** `AutomaticWithdrawalsAllowedChanged`
- **Args:**
    - **Holder:** Address of the token holder whose automatic withdrawals config was changed
    - **Allowed:** Whether automatic withdrawals are allowed or not for the given holder

##### 6.2.1.5. Module set

- **Name:** `ModuleSet`
- **Args:**
    - **Module ID:** ID of the module being set
    - **Address:** Address of the module being set

##### 6.2.1.6. Module enabled

- **Name:** `ModuleEnabled`
- **Args:**
    - **Module ID:** ID of the enabled module
    - **Address:** Address of the enabled module

##### 6.2.1.7. Module disabled

- **Name:** `ModuleDisabled`
- **Args:**
    - **Module ID:** ID of the disabled module
    - **Address:** Address of the disabled module

##### 6.2.1.8. Custom function set

- **Name:** `CustomFunctionSet`
- **Args:**
    - **Signature:** Signature of the function being set
    - **Target:** Address set as the target for the custom function

##### 6.2.1.9. Funds governor changed

- **Name:** `FundsGovernorChanged`
- **Args:**
    - **Previous governor:** Address of the previous funds governor
    - **Current governor:** Address of the current funds governor

##### 6.2.1.10. Config governor changed

- **Name:** `ConfigGovernorChanged`
- **Args:**
    - **Previous governor:** Address of the previous config governor
    - **Current governor:** Address of the current config governor

##### 6.2.1.11. Modules governor changed

- **Name:** `ModulesGovernorChanged`
- **Args:**
    - **Previous governor:** Address of the previous modules governor
    - **Current governor:** Address of the current modules governor

##### 6.2.1.12. Granted

- **Name:** `Granted`
- **Args:**
    - **Role:** ID of the role that was granted
    - **Who:** Address of the entity that was granted

##### 6.2.1.13. Revoked

- **Name:** `Revoked`
- **Args:**
    - **Role:** ID of the role that was revoked
    - **Who:** Address of the entity that was revoked

##### 6.2.1.14. Frozen

- **Name:** `Frozen`
- **Args:**
    - **Role:** ID of the role that was frozen

#### 6.2.2. Getters

The following functions are state getters provided by the `Controller`:

##### 6.2.2.3. Config

- **Inputs:**
    - **Term ID:** Identification number of the term querying the Protocol config of
- **Pre-flight checks:** None
- **Outputs:**
    - **Fee token:** Address of the token used to pay for fees
    - **Fees:** Array Array containing fee information:
        - **Guardian fee:** Amount of fee tokens that is paid per guardian per dispute
        - **Draft fee:** Amount of fee tokens per guardian to cover the drafting cost
        - **Settle fee:** Amount of fee tokens per guardian to cover round settlement cost
    - **Round state durations:** Array containing the durations in terms of the different phases of a dispute:
        - **Evidence terms:** Max submitting evidence period duration in Protocol terms
        - **Commit terms:** Commit period duration in Protocol terms
        - **Reveal terms:** Reveal period duration in Protocol terms
        - **Appeal terms:** Appeal period duration in Protocol terms
        - **Appeal confirmation terms:** Appeal confirmation period duration in Protocol terms
    - **Permyriads:** Array containing permyriads information:
        - **Penalty pct:** Permyriad of min active tokens balance to be locked for each drafted guardian (‱ - 1/10,000)
        - **Final round reduction:** Permyriad of fee reduction for the last appeal round (‱ - 1/10,000)
    - **Round params:** Array containing params for rounds:
        - **First round guardians number:** Number of guardians to be drafted for the first round of disputes
        - **Appeal step factor:** Increasing factor for the number of guardians of each round of a dispute
        - **Max regular appeal rounds:** Number of regular appeal rounds before the final round is triggered
        - **Final round lock terms:** Number of terms that a coherent guardian in a final round is disallowed to withdraw (to prevent 51% attacks)
    - **Appeal collateral params:** Array containing params for appeal collateral:
        - **Appeal collateral factor:** Multiple of dispute fees (guardians, draft, and settlements) required to appeal a preliminary ruling
        - **Appeal confirm collateral factor:** Multiple of dispute fees (guardians, draft, and settlements) required to confirm appeal

##### 6.2.2.4. Drafts config

- **Inputs:**
    - **Term ID:** Identification number of the term querying the Protocol drafts config of
- **Pre-flight checks:** None
- **Outputs:**
    - **Fee token:** ERC20 token to be used for the fees of the Protocol
    - **Draft fee:** Amount of fee tokens per guardian to cover the drafting cost
    - **Penalty pct:** Permyriad of min active tokens balance to be locked for each drafted guardian (‱ - 1/10,000)

##### 6.2.2.5. Minimum ANT active balance

- **Inputs:**
    - **Term ID:** Identification number of the term querying the Protocol min active balance of
- **Pre-flight checks:** None
- **Outputs:**
    - **Min active balance:** Minimum amount of guardian tokens guardians have to activate to participate in the Protocol

##### 6.2.2.6. Config change term ID

- **Inputs:** None
- **Pre-flight checks:** None
- **Outputs:**
    - **Config change term ID:** Term identification number of the next scheduled config change

##### 6.2.2.7. Term duration

- **Inputs:** None
- **Pre-flight checks:** None
- **Outputs:**
    - **Term duration:** Duration in seconds of the Protocol term

##### 6.2.2.8. Last ensured term ID

- **Inputs:** None
- **Pre-flight checks:** None
- **Outputs:**
    - **Last ensured term ID:** Identification number of the last ensured term

##### 6.2.2.9. Current term ID

- **Inputs:** None
- **Pre-flight checks:** None
- **Outputs:**
    - **Current term ID:** Identification number of the current term

##### 6.2.2.10. Needed transitions

- **Inputs:** None
- **Pre-flight checks:** None
- **Outputs:**
    - **Needed transitions:** Number of terms the Protocol should transition to be up-to-date

##### 6.2.2.11. Term

- **Inputs:**
    - **Term ID:** Identification number of the term being queried
- **Pre-flight checks:** None
- **Outputs:**
    - **Start time:** Term start time
    - **Randomness BN:** Block number used for randomness in the requested term
    - **Randomness:** Randomness computed for the requested term

##### 6.2.2.12. Term randomness

- **Inputs:**
    - **Term ID:** Identification number of the term being queried
- **Pre-flight checks:**
    - Ensure the term was already transitioned
- **Outputs:**
    - **Term randomness:** Randomness of the requested term

##### 6.2.2.13. Are withdrawals allowed for

- **Inputs:**
    - **Address:** Address of the token holder querying if withdrawals are allowed for
- **Pre-flight checks:** None
- **Outputs:**
    - **Allowed:** True if the given holder accepts automatic withdrawals of their tokens, false otherwise

##### 6.2.2.14. Funds governor

- **Inputs:** None
- **Pre-flight checks:** None
- **Outputs:**
    - **Funds governor:** Address of the funds governor

##### 6.2.2.15. Config governor

- **Inputs:** None
- **Pre-flight checks:** None
- **Outputs:**
    - **Config governor:** Address of the config governor

##### 6.2.2.16. Modules governor

- **Inputs:** None
- **Pre-flight checks:** None
- **Outputs:**
    - **Modules governor:** Address of the modules governor

##### 6.2.2.17. Is module active

- **Inputs:**
    - **Module ID:** ID of the module being queried
    - **Address:** Address of the module being queried
- **Pre-flight checks:**
    - Ensure that the given ID matches the ID of the requested module
- **Outputs:**
    - **Active:** Whether the requested module is active

##### 6.2.2.18. Module by address

- **Inputs:**
    - **Address:** Address of the module being queried
- **Pre-flight checks:** None
- **Outputs:**
    - **Module ID:** ID of the module associated to the given address
    - **Active:** Whether the requested module is active

##### 6.2.2.19. Module by ID

- **Inputs:**
    - **Module ID:** ID of the module being queried
- **Pre-flight checks:** None
- **Outputs:**
    - **Module address:** Address of the module queried

##### 6.2.2.20. Custom function

- **Inputs:**
    - **Signature:** Signature of the function being queried
- **Pre-flight checks:** None
- **Outputs:**
    - **Address:** Address of the target where the given signature will be forwarded

##### 6.2.2.21. Dispute Manager

- **Inputs:** None
- **Pre-flight checks:** None
- **Outputs:**
    - **Protocol address:** Address of the `DisputeManager` module set

##### 6.2.2.22. Guardians registry

- **Inputs:** None
- **Pre-flight checks:** None
- **Outputs:**
    - **Guardians registry address:** Address of the `GuardiansRegistry` module set

##### 6.2.2.23. Voting

- **Inputs:** None
- **Pre-flight checks:** None
- **Outputs:**
    - **Voting address:** Address of the `Voting` module set

##### 6.2.2.24. PaymentsBook

- **Inputs:** None
- **Pre-flight checks:** None
- **Outputs:**
    - **Payments book address:** Address of the `PaymentsBook` module set

##### 6.2.2.25. Treasury

- **Inputs:** None
- **Pre-flight checks:** None
- **Outputs:**
    - **Treasury address:** Address of the `Treasury` module set

##### 6.2.2.26. Has role

- **Inputs:**
    - **Who**: Address of the entity being queried
    - **Role**: ID of the role being queried
- **Pre-flight checks:** None
- **Outputs:**
    - **Has:** Whether the given entity has the requested role or not

##### 6.2.2.27. Is role frozen

- **Inputs:**
    - **Role**: ID of the role being queried
- **Pre-flight checks:** None
- **Outputs:**
    - **Frozen:** Whether the given role is frozen or not

### 6.3. Dispute Manager

#### 6.3.1 Events

The following events are emitted by the `DisputeManager`:

##### 6.3.1.1. New dispute

- **Name:** `NewDispute`
- **Args:**
    - **Dispute ID:** Identification number of the dispute that has been created
    - **Subject:** Address of the `Arbitrable` subject associated to the dispute
    - **Draft term ID:** Identification number of the term when the dispute will be able to be drafted
    - **Metadata:** Optional metadata that can be used to provide additional information on the created dispute

##### 6.3.1.2. Evidence period closed

- **Name:** `EvidencePeriodClosed`
- **Args:**
    - **Dispute ID:** Identification number of the dispute that has changed
    - **Term ID:** Term ID in which the dispute evidence period has been closed

##### 6.3.1.3. Guardian drafted

- **Name:** `GuardianDrafted`
- **Args:**
    - **Dispute ID:** Identification number of the dispute that was drafted
    - **Round ID:** Identification number of the dispute round that was drafted
    - **Guardian:** Address of the guardian drafted for the dispute

##### 6.3.1.4. Dispute changed

- **Name:** `DisputeStateChanged`
- **Args:**
    - **Dispute ID:** Identification number of the dispute that has changed
    - **State:** New dispute state: pre-draft, adjudicating, or ruled

##### 6.3.1.5. Ruling appealed

- **Name:** `RulingAppealed`
- **Args:**
    - **Dispute ID:** Identification number of the dispute appealed
    - **Round ID:** Identification number of the adjudication round appealed
    - **Ruling:** Ruling appealed in favor of

##### 6.3.1.6. Ruling appeal confirmed

- **Name:** `RulingAppealConfirmed`
- **Args:**
    - **Dispute ID:** Identification number of the dispute whose last round's appeal was confirmed
    - **Round ID:** Identification number of the adjudication round whose appeal was confirmed
    - **Draft term ID:** Identification number of the term when the next round will be able to be drafted

##### 6.3.1.7. Ruling computed

- **Name:** `RulingComputed`
- **Args:**
    - **Dispute ID:** Identification number of the dispute being ruled
    - **Ruling:** Final ruling decided for the dispute

##### 6.3.1.8. Penalties settled

- **Name:** `PenaltiesSettled`
- **Args:**
    - **Dispute ID:** Identification number of the dispute settled
    - **Round ID:** Identification number of the adjudication round settled
    - **Collected tokens:** Total amount of guardian tokens that were collected from slashed guardians for the requested round

##### 6.3.1.9. Reward settled

- **Name:** `RewardSettled`
- **Args:**
    - **Dispute ID:** Identification number of the dispute settled
    - **Round ID:** Identification number of the adjudication round settled
    - **Guardian:** Address of the guardian rewarded

##### 6.3.1.10. Appeal deposit settled

- **Name:** `AppealDepositSettled`
- **Args:**
    - **Dispute ID:** Identification number of the dispute whose round's appeal was settled
    - **Round ID:** Identification number of the adjudication round whose appeal was settled

##### 6.3.1.11. Max guardians per draft batch changed

- **Name:** `MaxGuardiansPerDraftBatchChanged`
- **Args:**
    - **Previous max guardians per draft batch:** Previous max number of guardians to be drafted per batch
    - **Current max guardians per draft batch:** New max number of guardians to be drafted per batch

#### 6.3.2. Getters

The following functions are state getters provided by the `DisputeManager`:

##### 6.3.2.1. Dispute fees

- **Inputs:** None
- **Pre-flight checks:** None
- **Outputs:**
    - **Fee token:** Address of the ERC20 token used for the fees
    - **Total fee:** Total amount of fees required to create a dispute in the next draft term

##### 6.3.2.2. Dispute

- **Inputs:**
    - **Dispute ID:** Identification number of the dispute being queried
- **Pre-flight checks:**
    - Ensure a dispute object with that ID exists
- **Outputs:**
    - **Subject:** Arbitrable subject being disputed
    - **Possible rulings:** Number of possible rulings allowed for the drafted guardians to vote on the dispute
    - **State:** Current state of the dispute being queried: pre-draft, adjudicating, or ruled
    - **Final ruling:** The winning ruling in case the dispute is finished
    - **Last round ID:** Identification number of the last round created for the dispute

##### 6.3.2.3. Round

- **Inputs:**
    - **Dispute ID:** Identification number of the dispute being queried
    - **Round ID:** Identification number of the adjudication round being queried
- **Pre-flight checks:**
    - Ensure a dispute object with that ID exists
    - Ensure an adjudication round object with that ID exists for the given dispute
- **Outputs:**
    - **Draft term ID:** Term from which the requested round can be drafted
    - **Delayed terms:** Number of terms the given round was delayed based on its requested draft term id
    - **Guardians number:** Number of guardians requested for the round
    - **Selected guardians:** Number of guardians already selected for the requested round
    - **Settled penalties:** Whether or not penalties have been settled for the requested round
    - **Collected tokens:** Amount of guardian tokens that were collected from slashed guardians for the requested round
    - **Coherent guardians:** Number of guardians that voted in favor of the final ruling in the requested round
    - **State:** Adjudication state of the requested round

##### 6.3.2.4. Appeal

- **Inputs:**
    - **Dispute ID:** Identification number of the dispute being queried
    - **Round ID:** Identification number of the adjudication round being queried
- **Pre-flight checks:**
    - Ensure a dispute object with that ID exists
    - Ensure an adjudication round object with that ID exists for the given dispute
- **Outputs:**
    - **Maker:** Address of the account appealing the given round
    - **Appealed ruling:** Ruling confirmed by the appealer of the given round
    - **Taker:** Address of the account confirming the appeal of the given round
    - **Opposed ruling:** Ruling confirmed by the appeal taker of the given round

##### 6.3.2.5. Next round details

- **Inputs:**
    - **Dispute ID:** Identification number of the dispute being queried
    - **Round ID:** Identification number of the adjudication round being queried
- **Pre-flight checks:**
    - Ensure a dispute object with that ID exists
    - Ensure an adjudication round object with that ID exists for the given dispute
- **Outputs:**
    - **Start term ID:** Term ID from which the next round will start
    - **Guardians number:** Guardians number for the next round
    - **New dispute state:** New state for the dispute associated to the given round after the appeal
    - **Fee token:** ERC20 token used for the next round fees
    - **Guardian fees:** Total amount of fees to be distributed between the winning guardians of the next round
    - **Total fees:** Total amount of fees for a regular round at the given term
    - **Appeal deposit:** Amount to be deposit of fees for a regular round at the given term
    - **Confirm appeal deposit:** Total amount of fees for a regular round at the given term

##### 6.3.2.6. Guardian

- **Inputs:**
    - **Dispute ID:** Identification number of the dispute being queried
    - **Round ID:** Identification number of the adjudication round being queried
    - **Guardian:** Address of the guardian being queried
- **Pre-flight checks:**
    - Ensure a dispute object with that ID exists
    - Ensure an adjudication round object with that ID exists for the given dispute
- **Outputs:**
    - **Weight:** Guardian weight drafted for the requested round
    - **Rewarded:** Whether or not the given guardian was rewarded based on the requested round

### 6.4. Guardians Registry

#### 6.4.1 Events

The following events are emitted by the `GuardiansRegistry`:

##### 6.4.1.1. Staked

- **Name:** `Staked`
- **Args:**
    - **Guardian:** Address of the guardian to stake the tokens to
    - **Amount:** Amount of tokens to be staked
    - **Total:** Total staked balance on the registry

##### 6.4.1.2. Unstaked

- **Name:** `Unstaked`
- **Args:**
    - **Guardian:** Address of the guardian to unstake the tokens of
    - **Amount:** Amount of tokens to be unstaked
    - **Total:** Total staked balance on the registry

##### 6.4.1.3. Guardian activated

- **Name:** `GuardianActivated`
- **Args:**
    - **Guardian:** Address of the guardian activated
    - **Amount:** Amount of guardian tokens activated
    - **From term ID:** Identification number of the term in which the guardian tokens will be activated

##### 6.4.1.4. Guardian deactivation requested

- **Name:** `GuardianDeactivationRequested`
- **Args:**
    - **Guardian:** Address of the guardian that requested a tokens deactivation
    - **Amount:** Amount of guardian tokens to be deactivated
    - **Available term ID:** Identification number of the term in which the requested tokens will be deactivated

##### 6.4.1.5. Guardian deactivation processed

- **Name:** `GuardianDeactivationProcessed`
- **Args:**
    - **Guardian:** Address of the guardian whose deactivation request was processed
    - **Amount:** Amount of guardian tokens deactivated
    - **Available term ID:** Identification number of the term in which the requested tokens will be deactivated
    - **Processed term ID:** Identification number of the term in which the given deactivation was processed

##### 6.4.1.6. Guardian deactivation updated

- **Name:** `GuardianDeactivationUpdated`
- **Args:**
    - **Guardian:** Address of the guardian whose deactivation request was updated
    - **Amount:** New amount of guardian tokens of the deactivation request
    - **Available term ID:** Identification number of the term in which the requested tokens will be deactivated
    - **Updated term ID:** Identification number of the term in which the given deactivation was updated

##### 6.4.1.7. Guardian activation lock changed

- **Name:** `GuardianActivationLockChanged`
- **Args:**
    - **Guardian:** Address of the guardian whose activation was changed
    - **Lock manager:** Address of the lock manager controlling the lock
    - **Amount:** New activation locked amount of the guardian
    - **Total:** New total activation lock of the guardian

##### 6.4.1.8. Guardian balance locked

- **Name:** `GuardianBalanceLocked`
- **Args:**
    - **Guardian:** Address of the guardian whose active balance was locked
    - **Amount:** New amount locked to the guardian

##### 6.4.1.9. Guardian balance unlocked

- **Name:** `GuardianBalanceUnlocked`
- **Args:**
    - **Guardian:** Address of the guardian whose active balance was unlocked
    - **Amount:** Amount of active locked that was unlocked to the guardian

##### 6.4.1.10. Guardian slashed

- **Name:** `GuardianSlashed`
- **Args:**
    - **Guardian:** Address of the guardian whose active tokens were slashed
    - **Amount:** Amount of guardian tokens slashed from the guardian active tokens
    - **Effective term ID:** Identification number of the term when the guardian active balance will be updated

##### 6.4.1.11. Guardian tokens assigned

- **Name:** `GuardianTokensAssigned`
- **Args:**
    - **Guardian:** Address of the guardian receiving tokens
    - **Amount:** Amount of guardian tokens assigned to the staked balance of the guardian

##### 6.4.1.12. Guardian tokens burned

- **Name:** `GuardianTokensBurned`
- **Args:**
    - **Amount:** Amount of guardian tokens burned to the zero address

##### 6.4.1.13. Guardian tokens collected

- **Name:** `GuardianTokensCollected`
- **Args:**
    - **Guardian:** Address of the guardian whose active tokens were collected
    - **Amount:** Amount of guardian tokens collected from the guardian active tokens
    - **Effective term ID:** Identification number of the term when the guardian active balance will be updated

##### 6.4.1.14. Total active balance limit changed

- **Name:** `TotalActiveBalanceLimitChanged`
- **Args:**
    - **Previous limit:** Previous total active balance limit
    - **Current limit:** Current total active balance limit

#### 6.4.2. Getters

The following functions are state getters provided by the `GuardiansRegistry`:

##### 6.4.2.1. Guardian token
- **Inputs:** None
- **Pre-flight checks:** None
- **Outputs:**
    - **Guardian token:** Address of the guardian token

##### 6.4.2.2. Total supply
- **Inputs:** None
- **Pre-flight checks:** None
- **Outputs:**
    - **Amount:** Total supply of guardian tokens staked

##### 6.4.2.3. Total active balance
- **Inputs:** None
- **Pre-flight checks:** None
- **Outputs:**
    - **Amount:** Total amount of active guardian tokens

##### 6.4.2.4. Total active balance at
- **Inputs:**
    - **Term ID:** Identification number of the term to query on
- **Pre-flight checks:** None
- **Outputs:**
    - **Amount:** Total amount of active guardian tokens at the given term ID

##### 6.4.2.5. Balance of
- **Inputs:**
    - **Guardian:** Address of the guardian querying the staked balance of
- **Pre-flight checks:** None
- **Outputs:**
    - **Amount:** Total balance of tokens held by a guardian

##### 6.4.2.6. Detailed balance of
- **Inputs:**
    - **Guardian:** Address of the guardian querying the detailed balance information of
- **Pre-flight checks:** None
- **Outputs:**
    - **Active:** Amount of active tokens of a guardian
    - **Available:** Amount of available tokens of a guardian
    - **Locked:** Amount of active tokens that are locked due to ongoing disputes
    - **Pending deactivation:** Amount of active tokens that were requested for deactivation

##### 6.4.2.7. Active balance of at
- **Inputs:**
    - **Guardian:** Address of the guardian querying the active balance of
    - **Term ID:** Identification number of the term to query on
- **Pre-flight checks:** None
- **Outputs:**
    - **Amount:** Amount of active tokens for guardian

##### 6.4.2.8. Unlocked active balance of
- **Inputs:**
    - **Guardian:** Address of the guardian querying the unlocked active balance of
- **Pre-flight checks:** None
- **Outputs:**
    - **Amount:** Amount of active tokens of a guardian that are not locked due to ongoing disputes

##### 6.4.2.9. Deactivation request
- **Inputs:**
    - **Guardian:** Address of the guardian querying the deactivation request of
- **Pre-flight checks:** None
- **Outputs:**
    - **Amount:** Amount of tokens to be deactivated
    - **Available term ID:** Term in which the deactivated amount will be available

##### 6.4.2.10. Activation lock
- **Inputs:**
    - **Guardian:** Address of the guardian querying the activation lock of
    - **Lock manager:** Address of the lock manager querying the activation lock of
- **Pre-flight checks:** None
- **Outputs:**
    - **Amount:** Lock activation amount controlled by the given lock manager
    - **Total:** Total activation lock for the given guardian

##### 6.4.2.11. Withdrawals lock term ID
- **Inputs:**
    - **Guardian:** Address of the guardian querying the lock term ID of
- **Pre-flight checks:** None
- **Outputs:**
    - **Term ID:** Term ID in which the guardian's withdrawals will be unlocked (due to final rounds)

##### 6.4.2.12. Total active balance limit
- **Inputs:** None
- **Pre-flight checks:** None
- **Outputs:**
    - **Total active balance limit:** Maximum amount of total active balance that can be held in the registry

##### 6.4.2.13. Guardian ID
- **Inputs:**
    - **Guardian:** Address of the guardian querying the ID of
- **Pre-flight checks:** None
- **Outputs:**
    - **Guardian ID:** Identification number associated to a guardian address, zero in case it wasn't registered yet

### 6.5. Voting

#### 6.5.1 Events

The following events are emitted by the `Voting`:

##### 6.5.1.1. Voting created

- **Name:** `VotingCreated`
- **Args:**
    - **Vote ID:** Identification number of the new vote instance that has been created
    - **Possible outcomes:** Number of possible outcomes of the new vote instance that has been created

##### 6.5.1.2. Vote committed

- **Name:** `VoteCommitted`
- **Args:**
    - **Vote ID:** Identification number of the vote instance where a vote has been committed
    - **Voter:** Address of the voter that has committed the vote
    - **Commitment:** Hashed outcome of the committed vote

##### 6.5.1.3. Vote revealed

- **Name:** `VoteRevealed`
- **Args:**
    - **Vote ID:** Identification number of the vote instance where a vote has been revealed
    - **Voter:** Address of the voter whose vote has been revealed
    - **Outcome:** Outcome of the vote that has been revealed

##### 6.5.1.4. Vote leaked

- **Name:** `VoteLeaked`
- **Args:**
    - **Vote ID:** Identification number of the vote instance where a vote has been leaked
    - **Voter:** Address of the voter whose vote has been leaked
    - **Outcome:** Outcome of the vote that has been leaked

##### 6.5.1.5. Delegate set

- **Name:** `DelegateSet`
- **Args:**
    - **Voter:** Address of the voter principal
    - **Delegate:** Address of the delegate 


#### 6.5.2. Getters

The following functions are state getters provided by the `Voting`:

##### 6.5.2.1. Max allowed outcome

- **Inputs:**
    - **Vote ID:** Vote identification number
- **Pre-flight checks:**
    - Ensure a vote object with that ID exists
- **Outputs:**
    - **Max outcome:** Max allowed outcome for the given vote instance

##### 6.5.2.2. Winning outcome

- **Inputs:**
    - **Vote ID:** Vote identification number
- **Pre-flight checks:**
    - Ensure a vote object with that ID exists
- **Outputs:**
    - **Winning outcome:** Winning outcome of the given vote instance or refused in case it's missing

##### 6.5.2.3. Outcome tally

- **Inputs:**
    - **Vote ID:** Vote identification number
    - **Outcome:** Outcome querying the tally of
- **Pre-flight checks:**
    - Ensure a vote object with that ID exists
- **Outputs:**
    - **Tally:** Tally of the outcome being queried for the given vote instance

##### 6.5.2.4. Is valid outcome

- **Inputs:**
    - **Vote ID:** Vote identification number
    - **Outcome:** Outcome to check if valid or not
- **Pre-flight checks:**
    - Ensure a vote object with that ID exists
- **Outputs:**
    - **Valid:** True if the given outcome is valid for the requested vote instance, false otherwise

##### 6.5.2.5. Voter outcome

- **Inputs:**
    - **Vote ID:** Vote identification number querying the outcome of
    - **Voter:** Address of the voter querying the outcome of
- **Pre-flight checks:**
    - Ensure a vote object with that ID exists
- **Outputs:**
    - **Outcome:** Outcome of the voter for the given vote instance

##### 6.5.2.6. Has voted in favor of

- **Inputs:**
    - **Vote ID:** Vote identification number querying if a voter voted in favor of a certain outcome
    - **Outcome:** Outcome to query if the given voter voted in favor of
    - **Voter:** Address of the voter to query if voted in favor of the given outcome
- **Pre-flight checks:**
    - Ensure a vote object with that ID exists
- **Outputs:**
    - **In favor:** True if the given voter voted in favor of the given outcome, false otherwise

##### 6.5.2.7. Voters in favor of

- **Inputs:**
    - **Vote ID:** Vote identification number querying if a voter voted in favor of a certain outcome
    - **Outcome:** Outcome to query if the given voter voted in favor of
    - **Voters:** List of addresses of the voters to be filtered
- **Pre-flight checks:**
    - Ensure a vote object with that ID exists
- **Outputs:**
    - **In favor:** List of results to tell whether a voter voted in favor of the given outcome or not

##### 6.5.2.8. Is delegate of

- **Inputs:**
    - **Voter:** Address of the guardian voting on behalf of
    - **Delegate:** Address of the delegate being queried
- **Pre-flight checks:** None
- **Outputs:**
    - **Allowed:** True if the given delegate currently represents the voter

### 6.6. PaymentsBook

#### 6.6.1 Events

The following events are emitted by the `PaymentsBook`:

##### 6.6.1.1. Payment received

- **Name:** `PaymentReceived`
- **Args:**
    - **Period ID:** Identification number of the payment period when the payment was received
    - **Payer:** Address paying on behalf of
    - **Token:** Address of the token used for the payment
    - **Amount:** Amount of tokens being paid
    - **Data:** Arbitrary data

##### 6.6.1.2. Guardian share claimed

- **Name:** `GuardianShareClaimed`
- **Args:**
    - **Period ID:** Identification number of the payment period claimed by the guardian
    - **Guardian:** Address of the guardian claiming their share
    - **Token:** Address of the token used for the share
    - **Amount:** Amount of tokens the guardian received for the requested period

##### 6.6.1.3. Governor share claimed

- **Name:** `GovernorShareClaimed`
- **Args:**
    - **Period ID:** Identification number of the payment period claimed by the governor
    - **Token:** Address of the token used for the share
    - **Amount:** Amount of tokens transferred to the governor address

##### 6.6.1.4. Governor share changed

- **Name:** `GovernorSharePctChanged`
- **Args:**
    - **Previous governor share:** Previous permyriad of collected payments that was being allocated to the governor
    - **Current governor share:** Current permyriad of collected payments that will be allocated to the governor

#### 6.6.2. Getters

The following functions are state getters provided by the `PaymentsBook`:

##### 6.6.2.1. Period duration

- **Inputs:** None
- **Pre-flight checks:** None
- **Outputs:**
    - **Duration:** Duration of a payment period in Protocol terms

##### 6.6.2.2. Governor share

- **Inputs:** None
- **Pre-flight checks:** None
- **Outputs:**
    - **Governor share:** Permyriad of collected payments that will be allocated to the governor of the Protocol (‱ - 1/10,000)

##### 6.6.2.3. Current period ID

- **Inputs:** None
- **Pre-flight checks:**
    - Ensure that the Protocol first term has already started
- **Outputs:**
    - **Period ID:** Identification number of the current period

##### 6.6.2.4. Period shares details

- **Inputs:**
    - **Period ID:** Identification number of the period being queried
    - **Token:** Address of the token being queried
- **Pre-flight checks:** None
- **Outputs:**
    - **Guardians share:** Total amount collected for the guardians during a period
    - **Governor share:** Total amount collected for the governor during a period

##### 6.6.2.5. Period balance details

- **Inputs:**
    - **Period ID:** Identification number of the period being queried
- **Pre-flight checks:** None
- **Outputs:**
    - **Balance checkpoint:** Protocol term ID of a period used to fetch the total active balance of the guardians registry
    - **Total active balance:** Total amount of guardian tokens active in the Protocol at the corresponding period checkpoint

##### 6.6.2.6. Guardian share

- **Inputs:**
    - **Period ID:** Identification number of the period being queried
    - **Guardian:** Address of the guardian querying the owed shared of
    - **Tokens:** List of addresses of the tokens being queried
- **Pre-flight checks:**
    - Ensure that the balance details of the requested period have been ensured
- **Outputs:**
    - **Amounts:** List of token amounts collected for the guardian in the given period

##### 6.6.2.7. Can guardian claim

- **Inputs:**
    - **Period ID:** Identification number of the period being queried
    - **Guardian:** Address of the guardian querying the owed shared of
    - **Tokens:** List of addresses of the tokens being queried
- **Pre-flight checks:** None
- **Outputs:**
    - **Claimed:** List of results considering true if the guardian's share can be claimed for the given period and token, false otherwise

##### 6.6.2.8. Governor share

- **Inputs:**
    - **Period ID:** Identification number of the period being queried
    - **Tokens:** List of addresses of the tokens being queried
- **Pre-flight checks:** None
- **Outputs:**
    - **Amounts:** List of token amount collected for the governor in the given period

##### 6.6.2.9. Can governor claim

- **Inputs:**
    - **Period ID:** Identification number of the period being queried
    - **Tokens:** List of addresses of the tokens being queried
- **Pre-flight checks:** None
- **Outputs:**
    - **Claimed:** List of results considering true if the governor's share can be claimed for the given period and token, false otherwise

### 6.7. Treasury

#### 6.7.1 Events

The following events are emitted by the `Treasury`:

##### 6.7.1.1. Assign

- **Name:** `Assign`
- **Args:**
    - **Token:** Address of the ERC20 token assigned
    - **From:** Address of the account that has deposited the tokens
    - **To:** Address of the account that has received the tokens
    - **Amount:** Number of tokens assigned to the recipient account

##### 6.7.1.2. Withdraw

- **Name:** `Withdraw`
- **Args:**
    - **Token:** Address of the ERC20 token withdrawn
    - **From:** Address of the account that has withdrawn the tokens
    - **To:** Address of the account that has received the tokens
    - **Amount:** Number of tokens withdrawn to the recipient account

#### 6.7.2. Getters

The following functions are state getters provided by the `Treasury`:

##### 6.7.2.1. Balance of

- **Inputs:**
    - **Token:** Address of the ERC20 token querying a holder's the balance of
    - **Holder:** Address of account querying the balance of
- **Pre-flight checks:** None
- **Outputs:**
    - **Balance:** Amount of tokens the holder owns

## 7. Additional documentation

The following documents complement the technical specification:

- Aragon v2 [blog post](https://aragon.org/blog/2)

The following documents describe Aragon Court, a pre-cursor to Aragon Protocol, and may also be useful for historical understanding:

- Aragon Network [white paper](https://github.com/aragon/whitepaper)
- Aragon Network [launch phases](https://forum.aragon.org/t/aragon-network-launch-phases-and-target-dates)
- Court valuation model [forum post](https://forum.aragon.org/t/ant-demand-modeling-framework/1389)
- Court v1 initial [forum post](https://forum.aragon.org/t/aragon-court-v1/691)
- Proposal agreements description [blog post](https://blog.aragon.one/proposal-agreements-and-the-aragon-court/)

## 8. Testing guide

This guide aims to cover all the things you should know in order to try Aragon Protocol or integrate your application with it.

### 8.1. Testing instances

There are a few testing instances already deployed for Aragon Protocol.
All of these are mimicking the mainnet instance with some exceptions of term durations to provide a better testing experience.
Additionally, all the instances are using their own deployed version of the following ERC20 tokens:
- ANT, the native token of Aragon Protocol. You will need some fake ANT to stake as a guardian to be selected to resolve disputes.
- DAI, used for the Aragon Protocol fees. You will need some fake DAI to pay the dispute fees.

Of course, there is an ERC20 faucet deployed for all these instances that you can use to claim some fake ANT or DAI to start testing. More information is outlined below on using these faucets.

#### 8.1.1. Staging

This is probably the most useful testing instance you would like to try.
Fees are low and protocol terms last a few minutes to make sure you can interact with it a bit faster.

- Network: Rinkeby
- Protocol term: 10 minutes
- Payment period: 3 protocol terms (30 minutes)
- Dashboard: https://protocol-staging.aragon.org/
- Address: [`0x52180af656a1923024d1accf1d827ab85ce48878`](http://rinkeby.etherscan.io/address/0x52180af656a1923024d1accf1d827ab85ce48878)
- Fake ANT: [`0x5bc9be34f98eb072696d63b5be5d4d2f2c03d0ad`](http://rinkeby.etherscan.io/address/0x5bc9be34f98eb072696d63b5be5d4d2f2c03d0ad)
- Fake DAI: [`0x3af6b2f907f0c55f279e0ed65751984e6cdc4a42`](http://rinkeby.etherscan.io/address/0x3af6b2f907f0c55f279e0ed65751984e6cdc4a42)
- ERC20 faucet: [`0x5561f73c3BBe8202F4D7E51aD2A1F22f1E056eFE`](http://rinkeby.etherscan.io/address/0x5561f73c3BBe8202F4D7E51aD2A1F22f1E056eFE)

#### 8.1.2. Rinkeby

This testing instance mirrors the instance deployed to Mainnet, same terms duration and fee amounts

- Network: Rinkeby
- Protocol term: 8 hours
- Payment period: 90 protocol terms (1 month)
- Dashboard: https://protocol-rinkeby.aragon.org/
- Address: [`0xe9180dBE762Fe39520fC9883f7f7EFeBA6506534`](http://rinkeby.etherscan.io/address/0xe9180dBE762Fe39520fC9883f7f7EFeBA6506534)
- Fake ANT: [`0x1FAB7d0D028ded72195322998003F6e82cF4cFdB`](http://rinkeby.etherscan.io/address/0x1FAB7d0D028ded72195322998003F6e82cF4cFdB)
- Fake DAI: [`0xe9a083d88eed757b1d633321ce0519f432c6284d`](http://rinkeby.etherscan.io/address/0xe9a083d88eed757b1d633321ce0519f432c6284d)
- ERC20 faucet: [`0x5561f73c3BBe8202F4D7E51aD2A1F22f1E056eFE`](http://rinkeby.etherscan.io/address/0x5561f73c3BBe8202F4D7E51aD2A1F22f1E056eFE)

#### 8.1.3. Ropsten

This testing instance basically mimics the Mainnet instance

- Network: Ropsten
- Protocol term: 8 hours
- Payment period: 90 protocol terms (1 month)
- Dashboard: https://protocol-ropsten.aragon.org/
- Address: [`0x3b26bc496aebaed5b3e0e81cde6b582cde71396e`](http://ropsten.etherscan.io/address/0x3b26bc496aebaed5b3e0e81cde6b582cde71396e)
- Fake ANT: [`0xc863e1ccc047beff17022f4229dbe6321a6bce65`](http://ropsten.etherscan.io/address/0xc863e1ccc047beff17022f4229dbe6321a6bce65)
- Fake DAI: [`0x4e1f48db14d7e1ada090c42ffe15ff3024eec8bf`](http://ropsten.etherscan.io/address/0x4e1f48db14d7e1ada090c42ffe15ff3024eec8bf)
- ERC20 faucet: [`0x83c1ECDC6fAAb783d9e3ac2C714C0eEce3349638`](http://ropsten.etherscan.io/address/0x83c1ECDC6fAAb783d9e3ac2C714C0eEce3349638)

#### 8.1.4. Local

> Unless you are familiar with using a local Aragon development environment, we recommend skipping ahead to Section 8.2 and using one of the other available testing instances (Staging/ Rinkeby/ Ropsten).

To deploy a local instance of Aragon Protocol you will need to clone the deployment scripts first:

```bash
git clone https://github.com/aragon/aragon-network-deploy/
cd aragon-network-deploy
npm i
```

Once you have done that, make sure you have a local Ganache running:

```bash
npx ganache-cli -i 15 --port 8545 --gasLimit 8000000 --deterministic
```

Then, open a separate terminal in the same directory of the scripts repo and deploy a local instance by running the following command:

```bash
npm run deploy:protocol --network ganache
```

This command will output the addresses of all the deployed modules of Aragon Protocol including the main entry point (the `AragonProtocol` contract).
Additionally, it should deploy a fake version of the ANT and DAI tokens usable for testing purposes as explained above.

### 8.2. Claiming fake tokens from the ERC20 faucets

You can claim ANT or DAI fake tokens from the ERC20 faucets.
You can do this directly through Etherscan, simply click in any of the faucet links shared above in section 8.1.
Once there, you just need to enable your Web3 account and call the `withdraw()` function providing the desired token address and amount:
![faucet](./faucet.png)

When claiming tokens remember to add the 18 zeroes for the decimals, for example 10 DAI should be requested as `10000000000000000000`.
Bear in mind there is a quota set for these faucets; they will only allow you to withdraw up to 10,000 fake-DAI or 10,000 fake-ANT every 7 days.

### 8.3. Installing the Aragon Protocol dev CLI tool

To interact with the deployed versions of Aragon Protocol, we built a node-based [CLI tool](https://github.com/aragon/protocol-backend/tree/development/packages/cli) that you can use.
Currently, there is no published version of it. But you can clone the GitHub repo and run it locally.
To continue with the testing guide you will need to use it. First, make sure you clone it and install its dependencies as follows:
```
git clone https://github.com/aragon/protocol-backend/
cd protocol-backend
git checkout master
yarn
cd packages/cli
```

This CLI tool is built on top of Truffle using a custom [config file](https://www.npmjs.com/package/@aragon/truffle-config-v5) provided by Aragon.
Please review that package's documentation to understand how to set up your private keys for testing.

Let's continue with the Aragon Protocol testing guide and see how we can use the CLI tool.

### 8.4. Becoming a guardian

To become a guardian you simply need to activate some ANT tokens into Aragon Protocol.
First make sure to have claimed some fake ANT tokens from the faucet corresponding to the Aragon Protocol instance you're willing to try.
For now, the testing instances require a minimum of 10,000 ANT so make sure to have at least that amount.
Then, you can activate tokens into Aragon Protocol using the `stake` and `activate` commands of the CLI tool as follows:

```bash
node ./bin/index.js stake --guardian [GUARDIAN] --amount [AMOUNT] --from [FROM] --network [NETWORK] --verbose
node ./bin/index.js activate --guardian [GUARDIAN] --amount [AMOUNT] --from [FROM] --network [NETWORK] --verbose
```

Where:
- `[GUARDIAN]`: address of the guardian you will activate the tokens for
- `[AMOUNT]`: amount of fake ANT tokens you will activate for the specified guardian (it doesn't require adding the decimals, so to activate 10,000 ANT simply enter `10000`)
- `[FROM]`: address paying for the fake ANT tokens; this must be the address you used to claim the tokens from the faucet
- `[NETWORK]`: name of the Aragon Protocol instance you are willing to use: `staging`, `rinkeby`, or `ropsten`

Note that you can also avoid the flag `--verbose` if you want to avoid having too much details about the transactions being sent to the network.

You can check your current stake as a guardian in the dashboards linked above in section 8.1.

### 8.5. Creating a dispute

As you may know, disputes can only be submitted to Aragon Protocol through smart contracts that implement a specific interface to support being ruled by the protocol itself.
This is specified by the [`IArbitrable`](../../packages/ethereum/contracts/arbitration/IArbitrable.sol) interface.

Thus, the first thing we should do is to deploy an Arbitrable contract. You can do this from the CLI running the following command:

```bash
node ./bin/index.js arbitrable -f [FROM] -n [NETWORK] --verbose
```

Where:
- `[FROM]`: address deploying the Arbitrable contract; this address will be the one available to create disputes with it
- `[NETWORK]`: name of the Aragon Protocol instance you are using: `staging`, `rinkeby`, or `ropsten`

This command will output the address of your new Arbitrable contract.

Now, we are almost ready to create a dispute. The last step is to send some fake DAI to the Arbitrable instance so that it can pay for the protocol's dispute fees.
The dispute fees are to pay the guardians for each dispute to be resolved.
For the testing instances, each dispute costs 30.87 fake-DAI (`30870000000000000000` with 18 decimals).
Thus, you will need to make a transfer from your account to your Arbitrable instance.
To do that you can use the Etherscan interface for the fake DAI instance linked in section 8.1.

Finally, we are ready to create your dispute running the following command:

```bash
node ./bin/index.js dispute \
  -a [ARBITRABLE] \
  -m [METADATA] \
  -e [EVIDENCE_1] [EVIDENCE_2] ... [EVIDENCE_N] \
  -s [SUBMITTER_1] [SUBMITTER_1] ... [SUBMITTER_N] \
  -c \
  -f [FROM] \
  -n [NETWORK] \
  --verbose
```

Where:
- `[ARBITRABLE]`: address of your Arbitrable instance
- `[METADATA]`: metadata to be linked for your dispute (continue reading to have a better understanding of how to build a proper dispute metadata)
- `[EVIDENCE_N]`: reference to a human-readable evidence (continue reading to have a better understanding of how to provide a proper evidence reference)
- `[SUBMITTER_N]`: addresses submitting each piece of evidence; this list should match the evidence list length
- `-c` flag: optional to declare that the evidence submission period should be immediately closed. Otherwise, you will need to manually close it afterwards.
- `[FROM]`: address owning the Arbitrable instance being called; this address must be the one you used to deploy the Arbitrable instance before
- `[NETWORK]`: name of the Aragon Protocol instance you are using: `staging`, `rinkeby`, or `ropsten`

This command will output the ID of the dispute you've just created.

A few things to bear in mind is that, even though the `[METADATA]` and `[EVIDENCE_N]` arguments could be any arbitrary information, in order to use the Protocol Dashboard to rule disputes, these should follow a specific structure.
Currently, the Protocol Dashboard supports reading these pieces of information from files hosted in IPFS. Thus, it expects the following formats:
- `[METADATA]`: `'{ "metadata": "[METADATA_CID]/metadata.json", "description": "Some dispute description" }'`
- `[EVIDENCE_N]`: `ipfs:[EVIDENCE_N_CID]`

Where `METADATA_CID` is the `CID` of a dir hosted in IPFS including a file `metadata.json`, and `[EVIDENCE_N_CID]` is the `CID` of a markdown file for the evidence #N hosted in IPFS.
Additionally, the `metadata.json` file must have the following structure:

```json
{
    "description": "[Your dispute description]",
    "agreementTitle": "[A title for your agreement file]",
    "agreementText": "[Path to the agreement file in the dir uploaded to IPFS]",
    "plaintiff": "[Ethereum address representing the plaintiff]",
    "defendant": "[Ethereum address representing the defendant]"
}
```

Even though `agreementTitle`, `agreementText`, `plaintiff` and `defendant` are optional values, you will have a much better experience if you provide those.

Additionally, it is recommended to upload all these pieces of information together to IPFS. For example, you can take a look at [these files](./sample-dispute) we used to create this [sample dispute](https://protocol-staging.aragon.org/disputes/15).
To upload those files we simply ran the following command while having the IPFS daemon running in background:

![ipfs](/ipfs-output.png)

As you can see, the `[METADATA_CID]` is the output marked in red, while the `[EVIDENCE_N_CID]`s are the ones in green.
Finally, following this example, this was the command we ran to create the dispute:

```bash
node ./bin/index.js dispute -a 0xe573D236d40F331d24420075Fb2EdE84B9968E3c -m '{ "metadata": "QmbN3uaqRMWypUGKUbjuhL8wCgFXGgfktQgoKhTp6zUm6o/metadata.json", "description": "Sample testing dispute" }' -e ipfs:QmQn1eK9jbKQtwHoUwgXw3QP7dZe6rSDmyPut9PLXeHjhR ipfs:QmWRBN26uoL7MdZJWhSuBaKgCVvStBQMvFwSxtseTDY32S -s 0x59d0b5475AcF30F24EcA10cb66BB5Ed75d3d9016 0x61F73dFc8561C322171c774E5BE0D9Ae21b2da42 -c -n staging --verbose
```

### 8.6. Ruling a dispute

You can use any of the Protocol Dashboard instances linked in section 8.1 to interact with your created disputes (note that in some environments, it may be difficult to ensure that your account is drafted due to the randomness nature of the protocol—and therefore can be difficult to come to a ruling you want).
If your dispute's metadata was not correctly formatted or made available as explained in sections 8.5.1 and 8.5.2, the dispute will most likely not display the intended information to guardians.

Alternatively, you can use the rest of the CLI tool [commands](https://github.com/aragon/protocol-backend/tree/master/packages/cli/#commands) to begin ruling your dispute:
- [`draft`](https://github.com/aragon/protocol-backend/blob/master/packages/cli/src/commands/draft.js): Draft dispute and close evidence submission period if necessary
- [`commit`](https://github.com/aragon/protocol-backend/blob/master/packages/cli/src/commands/commit.js): Commit vote for a dispute round
- [`reveal`](https://github.com/aragon/protocol-backend/blob/master/packages/cli/src/commands/reveal.js): Reveal committed vote
- [`appeal`](https://github.com/aragon/protocol-backend/blob/master/packages/cli/src/commands/appeal.js): Appeal dispute in favour of a certain outcome
- [`confirm-appeal`](https://github.com/aragon/protocol-backend/blob/master/packages/cli/src/commands/confirm-appeal.js): Confirm an existing appeal for a dispute
- [`execute`](https://github.com/aragon/protocol-backend/blob/master/packages/cli/src/commands/execute.js): Execute ruling for a dispute
