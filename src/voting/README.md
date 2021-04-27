# Voting

> Voting is the basis for governing any DAO. Consequently, voting is an integral part of
> the Aragon V2 governance system.

## Motivation

Voting is a tricky thing to get right. A good voting system has to solve fundamental
issues like trust and fairness. Thankfully, blockchain technology has given us the tools
to build trust-less and censorshipt resistant voting systems. But these tools come at a
price. Concretely, a voting solution for DAOs with optimistic governance faces the
following challenges:

- **Rising gas costs of transactions on the Ethereum chain.** With rising gas costs on
  Ethereum voting has become increasingly expensive. This negatively impacts voter
  turn-out and therefore makes governance less effective.
- **On-chain execution of voting results**. Aragon V2 is built around the idea of
  optimistic governance. This means that any voting process proposed by members of a DAO
  must be able to interact with smart contracts stored on the Ethereum chain.

In order to solve these challenges, Aragon V2 uses **Vocdoni Bridge**, an interface to
Vocdoni's voting infrastructure. In this section we will give a brief overview on how
Vocdoni Bridge solution integrates into the Aragon V2 system. For a more detailed
information on the architecture of the underlying voting infrastructure, please read the
following [article](https://blog.vocdoni.io/vocdoni-technical-overview-v1/). For a
detailed guide on how to use the Vocdoni Bridge for voting, please read the [Road to
Vote]().

## Vocdoni Bridge

### Introduction

The voting solution that powers the Vocdoni bridge can be characterized as an [Optimistic
Rollup](https://ethereum.org/en/developers/docs/layer-2-scaling/#optimistic-rollups). In
short, this means that voting is performed on a side chain, but that the results are
secured on the Ethereum chain. This moves most of the computation required to hold a vote
away from the Ethereum chain. Doing so therefore drastically reduces the gas price
required for a voting process.

> Note: Vocdoni's voting solution makes voting free for voters. Unfortunately, this does
> not mean that the entire voting process is free. The reason for this is that every
> interaction with the Ethereum chain requires a transaction and every transaction costs
> gas. In particular, registering the voting process on chain incurs gas costs. The
> advantage of Vocdoni's voting solution is that the number of interactions with the
> Ethereum chain is limited to registering the voting process. The casting of votes tied
> to that process is free.

Registering voting processes on the blockchain also allows voting processes to interact
with smart contracts on the Ethereum chain. In particular, this allows voting processes to
interact with other components of the Aragon V2 stack, such as Aragon Court.

> Note that in the context of Aragon V2, a voting process is called a proposal.

### Registering Tokens

Before a DAO can make use of Vocdoni's voting solution as part of Aragon V2, they are
required to [register their token](https://bridge.dev.vocdoni.net/tokens/add/) on the
Vocdoni Bridge.

### Voting Process

Once a token has been registered, any token holder can start a voting process. This can be
be divided into the following steps.

1. **Creation of a voting process.** On Vocdoni bridge, a token holder can navigate to the
   token's detail page, where they can start a voting process. Each voting process comes
   with a written explanation, the question on which members will vote as well as a set of
   possible answers for voters to choose form, and a voting period.
2. **Registering the voting process as proposal.** After successfully creating a problem,
   the process can be integrate into the V2 system. In order to do so, the process creator
   has to navigate to the process' detail page. There, a button labeled `Schedule` will
   redirect them to their DAO's console.
3. **Casting votes.** Token holders can then cast votes.

At this point, a voting process on the Vocdoni bridge has been tied to a proposal in the
Aragon V2 system. This means that this vote can now be challenged or can lead to the
execution of on-chain actions.
