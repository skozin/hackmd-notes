# Dual Governance mechanism design overview (2023-10-23)

**A version from 2023-10-23. Please find the current version [here](https://hackmd.io/@skozin/r17mlW2la).**

---

WIP by [sam](https://twitter.com/_skozin), [pshe](https://twitter.com/PsheEth), [kadmil](https://twitter.com/kadmil_eth), [sacha](https://twitter.com/sachayve), [Hasu](https://twitter.com/hasufl), [Izzy](https://twitter.com/IsdrsP), and [Vasiliy](https://twitter.com/_vshapovalov).

Currently, the Lido protocol governance consists of the Lido DAO that uses LDO voting to approve DAO proposals, along with an optimistic voting subsystem called Easy Tracks that is used for routine changes of low-impact parameters and falls back to LDO voting given any objection from LDO holders.

Additionally, there is a Gate Seal emergency committee that allows pausing certain protocol functionality (e.g. withdrawals) for a pre-configured amount of time sifficient for the DAO to vote on and execute a proposal. Gate Seal committee can only enact a pause once before losing its power (so it has to be re-elected by the DAO after that).

Dual governance mechanism is an iteration on the protocol governance that gives stakers a say by allowing them to block DAO decisions and providing a negotiation device between stakers and the DAO.

Another way of looking at dual governance is that it implements 1) a dynamic user-extensible timelock on DAO decisions and 2) a rage quit mechanism for stakers taking into account the specifics of how Ethereum withdrawals work.

## Definitions

* **Lido protocol:** code deployed on the Ethereum blockchain implementing:
    1. a middleware between the parties willing to delegate ETH for validating the Ethereum blockchain in exchange for staking rewards (stakers) and the parties willing to run Ethereum validators in exchange for a fee taken from staking rewards (node operators);
    2. a fungibility layer distributing ETH between node operators and issuing stakers a fungible deposit receipt token (stETH).
* **Protocol governance:** the mechanism allowing to change the Lido protocol parameters and upgrade non-ossified (mutable) parts of the protocol code.
* **LDO:** the fungible governance token of the Lido DAO.
* **Lido DAO:** code deployed on the Ethereum blockchain implementing a DAO that receives a fee taken from the staking rewards to its treasury and allows LDO holders to collectively vote on spending the treasury, changing parameters of the Lido protocol and upgrading the non-ossified parts of the Lido protocol code. Referred to as just **DAO** thoughout this document.
* **DAO proposal:** a specific change in the the onchain state of the Lido protocol or the Lido DAO proposed by LDO holders. Proposals have to be approved via onchain voting between LDO holders to become executable.
* **DAO decision:** a DAO proposal approved via onchain voting between LDO holders.
* **stETH:** the fungible deposit receipt token of the Lido protocol. Allows the holder to withdraw the deposited ETH plus all accrued rewards (minus the fees) and penalties. Rewards/penalties accrual is expressed by periodic rebases of the token balances.
* **wstETH:** an non-rebasable, immutable, and trustless wrapper around stETH deployed as an integral part of the Lido protocol. At any moment in time, there is a fixed wstETH/stETH rate effective for wrapping and unwrapping. The rate changes on each stETH rebase. _For the purpose of this document, any wstETH holder is considered an stETH holder with the stETH balance being equal wstETH balance multiplied by the current wstETH/stETH rate_.
* **Stakers:** EOAs and smart contract wallets that hold stETH and wstETH tokens or deposit them into various deFi protocols and ceFi platforms: DEXes, CEXes, lending and stablecoin protocols, custodies, etc.
* **Node operators:** parties registered in the Lido protocol willing to run Ethereum validators using the delegated ETH in exchange for a fee taken from the staking rewards. Node operators generate validator keys and at any time remain their sole holders, having full and exclusive control over Ethereum validators. Node operators are required to set their validators' withdrawal credentials to point to the specific Lido protocol smart contract.

## High-level description

The mechanism can be described as a state machine with each state imposing different limitations on the actions the DAO and stakers can perform.

![](https://hackmd.io/_uploads/HkLRFKI-p.png)

### Normal state

The Normal state is the state the governance is designed to spend the most time within. The DAO can publish proposals, vote on them and execute the decisions after the standard timelock of `GovernanceProposalTimelock` days.

At any point in time, stakers can signal their opposition to the DAO by moving their stETH into a dedicated smart contract called **veto signalling escrow**, as well as move their stETH out of this escrow. This creates an onchain oracle for measuring stakers' disagreement with the DAO decisions.

When a staker moves their stETH into the veto signalling escrow, they also provide a trustless proof of their stETH balance as of `VetoBalSnapshotShift` days ago. The **veto power** of the staker is set to the minimum of this past balance and the amount of stETH they have in the escrow. When a staker moves their stETH out of the veto signalling escrow, their veto power is re-evaluated as minimum of their current veto power and the amount of the staker's stETH left in the escrow.

> The past balance is considered to prevent an attack where a malicious actor exploits a smart contract vulnerability, mints unbacked stETH and withdraws it via the dual governance rage quit mechanism without the DAO having the time to observe the unbacked mints and deploy a fix.

At the moment the total veto power in the signalling escrow exceeds `VetoFirstSealThreshold` percent of the current stETH total supply, the governance is transferred into the Veto Signalling state.

> Proposed values, to be modeled and refined:
> `VetoBalSnapshotShift = 15 days`
> `VetoFirstSealThreshold = 1%`

### Veto Signalling state

The Veto Signalling state's purpose is two-fold:

1. Reduce information assymetry by allowing an active minority of stakers to block execution of a controversial DAO decision until it can be inspected and acted upon by the less active majority of stakers.
2. Provide a negotiation vehicle between stakers and the DAO.

While in this state, the DAO can vote on new proposals but cannot execute the decisions, including the decisions that were pending prior to the governance entering this state.

Stakers can freely move stETH in and out of the veto signalling escrow. Each time this happens, the target duration of the state is re-evaluated: from `VetoSignallingMinDuration` when the amount of veto power equals `VetoFirstSealThreshold` up to `VetoSignallingMaxDuration` when the amount of veto power is at least `VetoSecondSealThreshold`. If the amount of veto power is less than `VetoFirstSealThreshold`, the target duration is set to $0$.

$$
T_{target}(p) = 
\left\{ \begin{array}{lr}
    0, & \text{if } p \lt P_{min} \\
    T(p), & \text{if } P_{min} \leq p \leq P_{max} \\
    T_{max}, & \text{if } p \gt P_{max} \\
\end{array} \right.
$$

$$
T(p) = T_{min} + \frac{(p - P_{min})} {P_{max} - P_{min}} (T_{max} - T_{min})
$$

If, as the result of this re-evaluation or as the result of the time passing, the current duration of the state exceeds the target one, either of the following happens:

1. the Deactivation sub-state of the Veto Signalling state is entered if the combined veto power in the signalling escrow is below the `VetoSecondSealThreshold`,
2. otherwise, the Veto Signalling state is exited and the governance is transferred to the Rage Quit Accumulation state (this can happen iff the governance already spent `VetoSignallingMaxDuration` in this state).

There is one special kind of proposal that the DAO can both vote on and execute: $KillAllPendingProposals$. If this proposal passes and gets executed while the governance is in the Veto Signalling state, all proposals that are not executed at the moment the governance leaves the Veto Signalling state will become forever unexecutable (killed). This mechanism provides a way for the DAO and stakers to negotiate and de-escalate if consensus is reached.

> Proposed values, to be modeled and refined:
> `VetoSignallingMinDuration = 5 days`
> `VetoSignallingMaxDuration = 45 days`
> `VetoSecondSealThreshold = 10%`

#### Deactivation sub-state

The state's purpose is to allow all stakers to observe the Veto Signalling being deactivated and react accordingly before non-killed proposals can be executed.

In this sub-state, the DAO cannot submit and execute proposals but can vote on already submitted undecided proposals so they become decided.

If, as the result of a staker moving their stETH into the veto signalling escrow, the target duration of the Veto Signalling state becomes more than its current duration, the deactivation sub-state is exited (so only the main Veto Signalling state remains entered).

If this sub-state is not exited before `VetoSignallingDeactivationDuration` days pass since its entrance, the governance is transferred to the Veto Cooldown state. The `VetoSignallingDeactivationDuration` value should exceed the DAO voting duration to guarantee that no undecided DAO proposals are left by the time the Veto Cooldown is activated.

> Proposed values, to be modeled and refined:
> `VetoSignallingDeactivationDuration = 3 days`

### Veto Cooldown state

The Veto Cooldown state lasts `VetoCooldownDuration` days. In this state, the DAO cannot submit new proposals but can execute non-killed decisions. It exists to guarantee that no staker possessing `VetoFirstSealThreshold` stETH can lock the governance indefinitely without rage quitting the protocol.

If, by the time `VetoCooldownDuration` days pass, the total veto power in the signalling escrow exceeds `VetoFirstSealThreshold`, the governance is transferred to the Veto Signalling state. Otherwise, the governance is transferred to the Normal state.

> Proposed values, to be modeled and refined:
> `VetoCooldownDuration = 1 days`

### Rage Quit Accumulation state

Entering this state means that stakers and the DAO weren't able to resolve the dispute so the DAO is misaligned with a significant part of stakers.

Rage quit allows stakers to withdraw ETH without being subject to pending or new DAO decisions. The Rage Quit Accumulation state is the first phase of this process. In this state, the DAO can submit and vote on proposals but cannot execute any decisions.

Upon the Rage Quit Accumulaiton state entry, all stETH from the veto signalling escrow is moved to the **rage quit escrow**. This escrow doesn't allow withdrawing stETH: instead, all stETH locked there will be withdrawn to ETH as the result of the rage quit process.

This state gives additional time for stakers to observe the pending DAO decisions and sending stETH for withdrawal by locking it into the rage quit escrow.

The state lasts `RageQuitAccumulationDuration` days. After this time passes, the governance is transferred to the Rage Quit state.

> Proposed values, to be modeled and refined:
> `RageQuitAccumulationDuration = 30 days`

### Rage Quit state

The Rage Quit state allows all stakers that elected to leave the protocol via rage quit to fully withdraw their ETH without being subject to any new or pending DAO decisions.

Upon entry of the Rage Quit state, all stETH from the rage quit escrow is sent for withdrawal. This state lasts until the withdrawal is completed. The DAO can submit and vote on proposals but cannot execute the resulting decisions.

Stakers are not allowed to lock stETH into the rage quit escrow anymore. However, they can move stETH to the veto signalling escrow (see [Veto Signalling state](#Veto-Signalling-state)) to block pending DAO decisions by forcing the Veto Signalling as the next governance state.

When withdrawal is complete, two things happen simultaneously:

1. The withdrawn ETH is moved to the **rage quit withdrawal vault**, an immutable smart contract allowing all participants of the rage quit to get the withdrawn ETH after a timelock lasting `RageQuitEthWithdrawalTimelock` days.
2. The governance exits the Rage Quit state.

The next state depends on the total amount of veto power locked in the veto signalling escrow: if it exceeds `VetoFirstSealThreshold`, the governance is transferred to the Veto Signalling state; otherwise, to the Normal state.

> Proposed values, to be modeled and refined:
> `RageQuitEthWithdrawalTimelock = 60 days`

### Gate Seal behavior and the Tiebreaker Committee

If the Gate Seal emergency committee pauses any protocol functionality while the DAO is blocked from executing decisions by the dual governance mechanism, or if the DAO execution gets blocked while the Gate Seal-triggered pause is active, the pause lasts until the DAO execution is unblocked (in contrast to a fixed duration of the pause when the DAO functions normally).

If stETH minting, transfers, or withdrawals are paused while the governance is in the Rage Quit state, OR if the DAO execution is continuously blocked for more than `TieBreakerActivationTimeout`, the **Tiebreaker Committee** gets the power of executing any DAO decision by a supermajority vote within the committee.

The Tiebreaker Committee should consist of the following sub-committees, each representing a distinct interest group within the Ethereum community:

* Client teams.
* Ethereum Foundation.
* Lido Node Operators.
* Largest protocol DAOs (by TVL).

The execution of a DAO decision in the Rage Quit governance state should be approved by at least three of the four sub-committees. The approval by each sub-committee should require at least the majority support from its members. Each sub-committee should contain less than $1/4$ of the members that are also members of the Gate Seal committee.

The sub-committe members should be elected by a DAO vote (subject to dual governance) and reviewed at least every year.

> Proposed values, to be modeled and refined:
> `TieBreakerActivationTimeout = 1 year`

### Governance states overview

|State |DAO: submit props|DAO: kill props|DAO: exec props|stETH: join/leave signalling escrow|stETH: join rage quit escrow|
|------------------------------|---|---|---|---|---|
|Normal                        | ✓ | ✓ | ✓ | ✓ |   |
|Veto Signalling               | ✓ | ✓ |   | ✓ |   |
|Veto Signalling: deactivation |   | ✓ |   | ✓ |   |
|Veto Cooldown                 |   | ✓ | ✓ | ✓ |   |
|Rage Quit accumulation        | ✓ | ✓ |   |   | ✓ |
|Rage Quit                     | ✓ | ✓ |   | ✓ |   |

## On implementation

The Lido procotol uses role-based access model. Currently, all roles allowing to modify protocol parameters are assigned to either the DAO voting contracts or the Easy Track contracts, and all roles allowing to upgrade the protocol code or manage other roles are assigned solely to the DAO voting contracts.

We propose to implement dual governance as a proxy (call forwarder) between the DAO voting/Easy Track contracts and the protocol contracts. The proxy will forward calls resulting from execution of a specific DAO decision only if this execution is allowed based on the governance state.

Then, the DAO can gradually re-assign roles of the protocol contracts (together with their management rights) to the dual governance contracts, making it impossible for the DAO to bypass dual governance when executing decisions that require the re-assigned roles.

## Dual governance scope

Dual governance should cover any DAO decision that can potentially affect the protocol users, including:

* Upgrading, adding, and removing any protocol code.
* Changing the global protocol parameters and safety limits.
* Changing the parameters of:
  * The withdrawal queue.
  * The staking router, including addition and removal of staking modules.
  * Staking modules, including addition and removal of node operators within the curated staking module.
* Adding, removing, and replacing the oracle committee members.
* Adding, removing, and replacing the deposit security committee members.

Importantly, any change to the parameters of dual governance contracts should also be in the scope of dual governance.

Dual governance should not cover:

* Emergency actions triggered by curcuit-breaker multisigs and contracts. These actions should be limited in scope and time and shouldn't be able to change any protocol code.
* DAO decisions related to spending and managing the DAO treasury.

## Remaining questions

1. Can we protect from stETH minting vulnerability without considering the past balance?
2. Can we do without the Tiebreaker Committee to protect from an attacker abusing rage quit to exploit a SC vulnerability while locking the code upgradeability by the DAO?
3. Related to 2: is voting within stakers for a proposal execution while the DAO is locked better than an external committee? Allows to push a fix with collaboration from stakers but makes an attack on the DAO easier: you'd only need to bribe/purchase enough stake to outvote the active honest stakers. How about additionally requiring supermajority of NOs to support this vote? Successful vote may also delegate the execution right to an emergency committee instead of just executing, further limiting the committee power compared to the proposed design.


## Possible next iterations on governance and risk minimization

### More ways to activate veto

Assign veto power to node operators and/or client teams. For example, supermajority of node operators having the same power in activating the extended timelock and rage quit as $N\%$ of stETH (say, 10% so that only 5% of stETH is enough to trigger a rage quit).

### Extended timelocks on critical DAO decisions

Currently, all DAO decisions have the same static timelock.

Split all decision types into 2-3 groups by the potential impact on stakers and the network and assign different static timelock values to each of these groups, in addition to the dual governance user-activated dynamic timelock.

### Modular ossification

Modularize the protocol code so that stETH minting, transfers and withdrawals are detached ehough from the rest of the code to be either ossified or their upgrades heavily restricted.

Detaching the withdrawal flow may allow to significantly improve the Rage Quit state by only blocking the upgrades of parts of the code that can affect withdrawals processing. If these parts are formally verified on the bytecode level, this may allow to deprecate the Tiebreaker Committee and achieve better autonomy of the governance.

### Bytecode-level formal verification of minting and transfers

Verify a set of invariants related to stETH minting and transfers on the bytecode level (compared to the high-level code that's verified currently) so that minting unbacked stETH is either verifiably impossible or the unbacked proportion of every stETH minted is verifiably  strictly limited.

Affects DG by allowing to get rid of considering past stETH balance when joining veto.

### Non-token voting

Split the protocol governance into sub-categories, e.g. changes of security parameters, changes of validator set composition rules, upgrades of critical code, etc. For all or some categories, derive voting power from traits that cannot be easily purchased and that increase the probability of the actor 1) having the competency to assess the changes in this sub-category, 2) being incentives-aligned with the protocol and network users.

### Trustless ZK oracle

Currently, the state of the validators participating in the protocol (total balance, validator state, etc) is provided to the protocol smart contracts via an oracle committee elected by the DAO.

If this committee becomes malicious or dysfunctional, it might affect how and if withdrawals are processed, in the worst case leading to the governance being deadlocked in the Rage Quit state and requiring the intervention of the Tiebreaker Committee.

Replace this committee with a ZK-based trustless oracle for verifying consensus layer storage proofs and applying the corresponsing changes to the execution layer state.

Requires beacon state root being available on the execution layer (e.g. as proposed in [EIP-4788](https://eips.ethereum.org/EIPS/eip-4788)). Implementation of the first components of a trustless ZK oracle has already started: [1](https://research.lido.fi/t/zk-proof-total-value-locked-oracle/3726), [2](https://research.lido.fi/t/zkllvm-trustless-zk-proof-tvl-oracle/5028), [3](https://research.lido.fi/t/dendreth-a-trustless-oracle-for-liquid-staking-protocols/5136).

### [Adaptive quorum biasing][adaptive-qb]

[adaptive-qb]: https://polkassembly.medium.com/adaptive-quorum-biasing-9b7e6d2a2261

Make the relative support required for an LDO vote to pass dependent on the total participation. Allow a proposal to be approved by a simple majority of the participating LDO voting power only given a significant participation (e.g. more than 30%); require a supermajority of the participating LDO voting power otherwise.

This reduces the probability of a minority of LDO voters approving a contentious DAO proposal and increases the potential cost of an attack on the DAO.

### Invariant-based circuit breaker

Implement an immutable smart contract allowing anyone to trustlessly prove that any of the critical protocol invariants are broken by a state transition, e.g.:

* stETH was minted without providing the corresponding ETH amount;
* ETH was withdrawn without burning the corresponding stETH amount;
* stETH total shares amount was changed outside of a mint/burn;
* etc.

Providing a proof of an incorrect state transition transfers the protocol into an emergency mode, disabling or restricting the affected functionality and potentially changing the governance operation mode.

### WC and LST forking

Allow detaching a portion of the protocol's validators so that their withdrawal credentials smart contract, along with any auxiliary code (e.g. withdrawal queue) is not managed by the DAO anymore: either ossify the WC or pass the administrative rights to a new DAO.

Withdrawal mechanism detachment and ossification would allow performing rage quits without extended and externally-dependent lock of the governance. Detachment followed by passing administrative rights may eventually be used as a component of the proper full-protocol forking mechanism (though coordinating between stakers and node operators will not be trivial).

Any flavor of WC forking will most probably require the consensus layer support of WC rotation between the same WC type.

### DAO voter bonding

If a DAO decision gets opposed by stakers who trigger the dual governance Veto Signalling state, require DAO participants to put up an LDO bond proportional to the stakers opposition in order for the decision to remain executable if the opposition doesn't lead to a rage quit and the DAO doesn't kill the decision. Then, if a rage quit still happens or the decision is killed, the bonded LDO is either burned or jailed, allowing the DAO to later decide is fate via a supermajority vote.

The escalation game is rougly this:

* DAO approves a decision, the timelock starts.
* Enough stakers oppose the decision to trigger Veto Signalling (an extended timelock).
* During this timelock, any LDO holder can put some of their LDO as a bond for a particular decision. The more stETH supply joins the opposition, the larger cumulative bond is required. The minimum cumulative bond should probably exceed the quorum value.
* Have LDO holders put up a bond large enough to overcome the opposition by the end of Veto Signalling?
  * No ⇒
    * The decision gets automatically killed after the Veto Signalling ends, regardless of the opposition strength.
  * Yes ⇒
    * Is the decision killed by the DAO **OR** does opposition reach the rage quit level by the end of Veto Signalling?
      * No ⇒
          * LDO bond is released.
          * The decision can be executed in the Veto Cooldown.
      * Yes ⇒
          * LDO bond gets burnt/jailed.
          * The decision gets killed if not killed already.
          * Rage quit starts.

A variation: require DAO members to also provide an ETH or stETH bond in addition to the LDO bond.

This mechanism increases the cost of a vote-bying attack on the DAO by introducing an additional skin in the game: the attacker now has to provide a bribe large enough to offset not only the (probability-weighted) price depreciation in the case the vote succeeds but also the complete or partial loss of the tokens in the case the vote is successfully opposed by users or other DAO members.

## Scenario analysis (TODO)

* Rouge or misaligned DAO minority
* Full DAO capture
* Collusion between Node Operators
* stETH minting vuln exploit
* stETH freezing the DAO
* stETH freezing the DAO to exploit a SC vuln
* Slow degradation of the social contract
* ...

## Changelog

### 2023-10-23 (this document)

A major re-work of the DG mechanism (previous version [here](https://hackmd.io/@lido/BJKmFkM-i)).

* Replaced the global settlement mechanism with the rage quit mechanism (i.e. local settlement, a protected exit from the protocol). Removed states: Global Settlement; added states: Rage Quit Accumulation, Rage Quit.
* Removed the veto lift voting mechanism.
* Re-worked the DG activation and negotiation mechanism, replacing Veto Voting and Veto Negotiation states with the Veto Signalling state.
* Added the Veto Cooldown state.
* Added the $KillAllPendingProposals$ DAO decision.
* Added stETH balance snapshotting mechanism.
* Specified inter-operation between the Gate Seal mechanism and DG.
* Added the Tiebreaker Committee.