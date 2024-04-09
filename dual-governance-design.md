# Dual Governance mechanism design overview (2024-03-22)

**A version from 2024-03-22. Please find the current version [here](https://hackmd.io/@skozin/r17mlW2la).**

---

WIP by [sam](https://twitter.com/_skozin), [pshe](https://twitter.com/PsheEth), [kadmil](https://twitter.com/kadmil_eth), [sacha](https://twitter.com/sachayve), [Hasu](https://twitter.com/hasufl), [Izzy](https://twitter.com/IsdrsP), and [Vasiliy](https://twitter.com/_vshapovalov).

Currently, the Lido protocol governance consists of the Lido DAO that uses LDO voting to approve DAO proposals, along with an optimistic voting subsystem called Easy Tracks that is used for routine changes of low-impact parameters and falls back to LDO voting given any objection from LDO holders.

Additionally, there is a Gate Seal emergency committee that allows pausing certain protocol functionality (e.g. withdrawals) for a pre-configured amount of time sifficient for the DAO to vote on and execute a proposal. Gate Seal committee can only enact a pause once before losing its power (so it has to be re-elected by the DAO after that).

Dual governance mechanism (DG) is an iteration on the protocol governance that gives stakers a say by allowing them to block DAO decisions and providing a negotiation device between stakers and the DAO.

Another way of looking at dual governance is that it implements 1) a dynamic user-extensible timelock on DAO decisions and 2) a rage quit mechanism for stakers taking into account the specifics of how Ethereum withdrawals work.

## Definitions

* **Lido protocol:** code deployed on the Ethereum blockchain implementing:
    1. a middleware between the parties willing to delegate ETH for validating the Ethereum blockchain in exchange for staking rewards (stakers) and the parties willing to run Ethereum validators in exchange for a fee taken from staking rewards (node operators);
    2. a fungibility layer distributing ETH between node operators and issuing stakers a fungible deposit receipt token (stETH).
* **Protocol governance:** the mechanism allowing to change the Lido protocol parameters and upgrade non-ossified (mutable) parts of the protocol code.
* **LDO:** the fungible governance token of the Lido DAO.
* **Lido DAO:** code deployed on the Ethereum blockchain implementing a DAO that receives a fee taken from the staking rewards to its treasury and allows LDO holders to collectively vote on spending the treasury, changing parameters of the Lido protocol and upgrading the non-ossified parts of the Lido protocol code. Referred to as just **DAO** thoughout this document.
* **DAO proposal:** a specific change in the the onchain state of the Lido protocol or the Lido DAO proposed by LDO holders. Proposals have to be approved via onchain voting between LDO holders to become executable.
* **stETH:** the fungible deposit receipt token of the Lido protocol. Allows the holder to withdraw the deposited ETH plus all accrued rewards (minus the fees) and penalties. Rewards/penalties accrual is expressed by periodic rebases of the token balances.
* **wstETH:** an non-rebasable, immutable, and trustless wrapper around stETH deployed as an integral part of the Lido protocol. At any moment in time, there is a fixed wstETH/stETH rate effective for wrapping and unwrapping. The rate changes on each stETH rebase.
* **Withdrawal NFT:** a non-fungible token minted by the Lido withdrawal queue contract as part of the (w)stETH withdrawal to ETH, parametrized by the underlying (w)stETH amount and the position in queue. Gives holder the right to claim the corresponding ETH amount after the withdrawal is complete. Doesn't entitle the holder to receive staking rewards.
* **Stakers:** EOAs and smart contract wallets that hold stETH, wstETH tokens and withdrawal NFTs or deposit them into various deFi protocols and ceFi platforms: DEXes, CEXes, lending and stablecoin protocols, custodies, etc.
* **Node operators:** parties registered in the Lido protocol willing to run Ethereum validators using the delegated ETH in exchange for a fee taken from the staking rewards. Node operators generate validator keys and at any time remain their sole holders, having full and exclusive control over Ethereum validators. Node operators are required to set their validators' withdrawal credentials to point to the specific Lido protocol smart contract.

## Mechanism description

### Proposal lifecycle

The DG assumes that any permissions [protected by the subsystem](#Dual-governance-scope) (which we will call the in-scope permissions) are assigned to the DG contracts, in contrast to being assigned to the DAO voting systems. Thus, it's impossible for the DAO to execute any in-scope changes bypassing the DG.

Instead of making the in-scope changes directly, the DAO voting script should submit them as a proposal to the DG subsystem upon execution of the approved DAO vote.

![image](https://hackmd.io/_uploads/r1Gtik5Ra.png)

After submission to the DG, a proposal can exist in one of the following states:

* **Pending**: a proposal approved by the DAO was submitted to the DG subsystem, starting the dynamic execution timelock.
* **Cancelled**: the DAO votes for cancelling the pending proposal. This is the terminal state.
* **Executed**: the dynamic timelock of a pending proposal has elapsed and the proposal was executed. This is the terminal state.


### Signalling Escrow

At any point in time, stakers can signal their opposition to the DAO by locking their stETH, wstETH, and [unfinalized](https://docs.lido.fi/contracts/withdrawal-queue-erc721#finalization) withdrawal NFTs into a dedicated smart contract called **veto signalling escrow**, as well as lift this signal by unlocking the tokens from the escrow. This creates an onchain oracle for measuring stakers' disagreement with the DAO decisions.

While stETH or wstETH tokens are locked in the signalling escrow, they still generate staking rewards and are still subject to potential slashings.

An address having stETH or wstETH locked in the signalling escrow can trigger an immediate withdrawal of the locked tokens to ETH while keeping the resulting withdrawal NFT locked in the signalling escrow.

Let's define the **rage quit support** $R$ as a dimensionless quantity calculated as follows:

$$
R = \frac{1}{ S_{st} + \text{eth}_f } \left(
  \text{st} + \text{eth}_f +
  R_{wst}^{st} ( \text{wst} + \text{shares}_u )
\right)
$$

$$
\text{eth}_f = \sum_{N_i \in \text{WR}_f} \text{eth}(N_i)
$$

$$
\text{shares}_u = \sum_{N_i \in \text{WR}_u} \text{shares}(N_i)
$$

where

* $S_{st}$ is the current stETH total supply,
* $\text{st}$ is the total amount of stETH locked in the signalling escrow,
* $R_{wst}^{st}$ is the current conversion rate from wstETH to stETH (the result of the `stETH.getPooledEthByShares(10**18) / 10**18` call), 
* $\text{wst}$ is total amount of wstETH locked in the signalling escrow,
* $\text{WR}_f$ is the set of finalized withdrawal NFTs locked in the signalling escrow,
* $\text{WR}_u$ is the set of non-finalized withdrawal NFTs locked in the escrow,
* $\text{shares}(N_i)$ is the stETH shares amount corresponding to the unfinalized withdrawal NFT $N_i$, 
* $\text{eth}(N_i)$ is the withdrawn ETH amount associated with the finalized withdrawal NFT $N_i$.

Changes in the rage quit support act as the main driver for the global governance state transitions.


### Global governance state

The DG mechanism can be described as a state machine defining the global governance state, with each particular state imposing different limitations on the actions the DAO can perform, and state transitions being driven by stakers' actions and (w)stETH withdrawals processing.

|State |DAO can submit proposals|DAO can execute proposals|
|-------------------------------|---|---|
|Normal                         | ✓ | ✓ |
|Veto Signalling                | ✓ |   |
|Veto Signalling (deactivation) |   |   |
|Veto Cooldown                  |   | ✓ |
|Rage Quit                      | ✓ |   |

![image](https://hackmd.io/_uploads/SkhB7bqRp.png)

Let's now define these states and transitions.


### Normal state

The Normal state is the state the mechanism is designed to spend the most time within. The DAO can submit the approved proposals to the DG and execute them after the standard timelock of `ProposalExecutionMinTimelock` days.

If, while the state is active, the [rage quit support](#Signalling-Escrow) exceeds the `FirstSealRageQuitSupport`, the governance is transferred into the Veto Signalling state.

```env
# Proposed values, to be modeled and refined
ProposalExecutionMinTimelock = 3 days
FirstSealRageQuitSupport = 0.01
```


### Veto Signalling state

The Veto Signalling state's purpose is two-fold:

1. Reduce information assymetry by allowing an active minority of stakers to block the execution of a controversial DAO decision until it can be inspected and acted upon by the less active majority of stakers.
2. Provide a negotiation vehicle between stakers and the DAO.

In this state, the DAO can submit approved proposals to the DG but cannot execute them, including the proposals that were pending prior to the governance entering this state, effectively extending the timelock on all such proposals.

The only proposal that can be executed by the DAO is the special $CancelAllPendingProposals$ action that cancels all proposals that were pending at the moment of this execution, making them forever unexecutable. This mechanism provides a way for the DAO and stakers to negotiate and de-escalate if consensus is reached.

The target duration $T^S_{target}$ of the state depends on the current rage quit support $R$ and can be calculated as follows:

$$
T^S_{target}(R) = 
\left\{ \begin{array}{lr}
    0, & \text{if } R \lt R_1 \\
    T^S(R), & \text{if } R_1 \leq R \leq R_2 \\
    T^S_{max}, & \text{if } R \gt R_2 \\
\end{array} \right.
$$

$$
T^S(R) = T^S_{min} + \frac{(R - R_1)} {R_2 - R_1} (T^S_{max} - T^S_{min})
$$

where $R_1$ is `FirstSealRageQuitSupport`, $R_2$ is `SecondSealRageQuitSupport`, $T^S_{min}$ is `VetoSignallingMinDuration`, $T^S_{max}$ is `VetoSignallingMaxDuration`. The dependence of the target duration on the rage quit support can be illustrated by the following graph:

![image](https://hackmd.io/_uploads/rJnj1LoR6.png)

When the current rage quit support changes due to stakers locking or unlocking tokens into/out of the signalling escrow or the total stETH supply changing, the target duration is re-evaluated. If, as the result of this re-evaluation or as the result of the time passing, the current duration of the state exceeds the target one, either of the following happens:

1. if the rage quit support is below the `SecondSealRageQuitSupport`, the Deactivation sub-state of the Veto Signalling state is entered without exiting the parent Veto Signalling state,
2. otherwise, the Veto Signalling state is exited and the governance is transferred to the Rage Quit state (this can happen iff the governance has already spent `VetoSignallingMaxDuration` in this state).

```env
# Proposed values, to be modeled and refined
VetoSignallingMinDuration = 5 days
VetoSignallingMaxDuration = 45 days
SecondSealRageQuitSupport = 0.1
```

#### Deactivation sub-state

The sub-state's purpose is to allow all stakers to observe the Veto Signalling being deactivated and react accordingly before non-cancelled proposals can be executed. In this sub-state, the DAO cannot submit proposals to the DG or execute pending proposals.

Since this is a sub-state, the time it's being active counts towards the parent Veto Signalling state duration.

The maximum time this sub-state can remain active, $T^D_{max}$, is calculated at the moment it gets entered as follows: if there were no proposals submitted to the DG since the last activation of the Veto Signalling state, then 

$$
T^D_{max} = T^D_{min}
$$

where $T^D_{min}$ is `VetoSignallingDeactivationMinDuration`. Otherwise, it's defined by the following expression:

$$
T^D_{max} = \max \left\{ T^D_{min}, \; t_{prop} + T^S_{max} - t \right\}
$$

where $t_{prop}$ is the moment the last proposal was submitted to the DG, $T^S_{max}$ is `VetoSignallingMaxDuration`, $t$ is the current time, i.e. the moment the Deactivation state is entered at.

If the current duration of the Deactivation state becomes larger than $T^D_{max}$, the Deactivation sub-state is exited along with its parent Veto Signalling state and the governance is transferred to the Veto Cooldown state.

If, while the sub-state is active and as the result of the rage quit support changing, the target duration of the Veto Signalling state becomes more than its current duration (which includes the time the Deactivation sub-state is being active), the Deactivation sub-state is exited so only the main Veto Signalling state remains active.

If, while the sub-state is active, the rage quit support exceeds the `SecondSealRageQuitSupport` AND the current duration of the Veto Signalling state exceeds the `VetoSignallingMaxDuration`, the Deactivation sub-state and its parent Veto Signalling state are exited and the governance is transferred to the Rage Quit state.

```env
# Proposed values, to be modeled and refined
VetoSignallingDeactivationMinDuration = 3 days
```

### Veto Cooldown state

In the Veto Cooldown state, the DAO cannot submit proposals to the DG but can execute pending non-cancelled proposals. It exists to guarantee that no staker possessing `FirstSealRageQuitSupport` stETH can lock the governance indefinitely without rage quitting the protocol.

The state duration is fixed at `VetoCooldownDuration`. After this time passes since the state activation, the state is exited and the governance is transferred either to the Normal state (if the rage quit support at that moment is less than `FirstSealRageQuitSupport`) or to the Veto Signalling state (otherwise).

```env
# Proposed values, to be modeled and refined
VetoCooldownDuration = 1 days
```


### Rage Quit state

The Rage Quit state allows all stakers that elected to leave the protocol via rage quit to fully withdraw their ETH without being subject to any new or pending DAO decisions. Entering this state means that stakers and the DAO weren't able to resolve the dispute so the DAO is misaligned with a significant part of stakers.

Upon entry of the Rage Quit state, three things happen:

1. The veto signalling escrow is irreversibly transformed into the **rage quit escrow**, an immutable smart contract that holds all tokens that are the part of the rage quit withdrawal process, i.e. stETH, wstETH, withdrawal NFTs, and the withdrawn ETH, and allows stakers to claim the withdrawn ETH after a certain timelock following the completion of the withdrawal process (with the timelock being determined at the moment of the Rage Quit state entry).
2. All stETH and wstETH held by the rage quit escrow is sent for withdrawal via the regular Lido Withdrawal Queue mechanism, generating a set of batch withdrawal NFTs held by the rage quit escrow.
3. A new instance of the veto signalling escrow smart contract is deployed. This way, at any point in time, there is only one veto signalling escrow but there may be multiple rage quit escrows from previous rage quits.

In this state, the DAO is allowed submit proposals to the DG but cannot execute any pending proposals. Stakers are not allowed to lock (w)stETH or withdrawal NFTs into the rage quit escrow so joining the ongoing rage quit is not possible. However, they can lock their tokens that are not part of the ongoing rage quit process to the newly-deployed veto signalling escrow to potentially trigger a new rage quit later.

The state lasts until the withdrawal started in 2) is complete, i.e. until all batch withdrawal NFTs generated from (w)stETH that was locked in the escrow are fullfilled and claimed, plus `RageQuitExtensionDelay` days. 

If, prior to the Rage Quit state being entered, a staker locked a withdrawal NFT into the signalling escrow, this NFT remains locked in the rage quit escrow. When such an NFT becomes fullfilled, the staker is allowed to burn this NFT and convert it to plain ETH, although still locked in the escrow. This allows stakers to derisk their ETH as early as possible by removing any dependence on the DAO decisions (remember that the withdrawal NFT contract is potentially upgradeable by the DAO but the rage quit escrow is immutable).

Since batch withdrawal NFTs are generated after the NFTs that were locked by stakers into the escrow directly, the withdrawal queue mechanism guarantees that, by the time batch NFTs are fullfilled, all individually locked NFTs are fullfilled as well and can be claimed. Together with the extension delay, this guarantees that any staker having a withdrawal NFT locked in the rage quit escrow have at least `RageQuitExtensionDelay` days to convert it to escrow-locked ETH before the DAO execution is unblocked.

When the withdrawal is complete and the extension delay elapses, two things happen simultaneously:

1. A timelock lasting $W(i)$ days is started, during which the withdrawn ETH remains locked in the rage quit escrow. After the timelock elapses, stakers who participated in the rage quit can obtain their ETH from the rage quit escrow.
2. The governance exits the Rage Quit state.

The next state depends on the current rage quit support (which depends on the amount of tokens locked in the veto signalling escrow deployed in 3): if it exceeds `FirstSealRageQuitSupport`, the governance is transferred to the Veto Signalling state; otherwise, to the Normal state.

The duration of the ETH claim timelock $W(i)$ is a non-linear function that depends on the rage quit sequence number $i$ (see below):

$$
W(i) = W_{min} +
\left\{ \begin{array}{lc}
    0, & \text{if } i \lt i_{min} \\
    g_W(i - i_{min}), & \text{otherwise} \\
\end{array} \right.
$$

where $W_{min}$ is `RageQuitEthClaimMinTimelock`, $i_{min}$ is `RageQuitEthClaimTimelockGrowthStartSeqNumber`, and $g_W(x)$ is a quadratic polynomial function with coefficients `RageQuitEthClaimTimelockGrowthCoeffs` (a list of length 3).

The rage quit sequence number is calculated as follows: each time the Normal state is entered, the sequence number is set to 0; each time the Rage Quit state is entered, the number is incremented by 1.

```env
# Proposed values, to be modeled and refined
RageQuitExtensionDelay = 7 days
RageQuitEthClaimMinTimelock = 60 days
RageQuitEthClaimTimelockGrowthStartSeqNumber = 2
RageQuitEthClaimTimelockGrowthCoeffs = (0, TODO, TODO)
```

### Gate Seal behavior and Tiebreaker Committee

The [Gate Seal](https://docs.lido.fi/contracts/gate-seal) is an existing curcuit breaker mechanism designed to be activated in the event of a zero-day vulnerability in the protocol contracts being found or exploited and empowering a DAO-elected committee to pause certain protocol functionality, including withdrawals, for a predefined duration enough for the DAO to vote for and execute a remediation. When this happens, the committee immediately loses its power. If this never happens, the committee's power also expires after a pre-configured amount of time passes since its election.

The pre-defined pause duration currently works since all DAO proposals have a fixed execution timelock so it's possible to configure the pause in a way that would ensure the DAO has enough time to vote on a fix, wait until the execution timelock expires, and execute the proposal before the pause ends.

The DG mechanism introduces a dynamic timelock on DAO proposals dependent on stakers' actions and protocol withdrawals processing which, in turn, requires making the Gate Seal pause duration also dynamic for the Gate Seal to remain an efficient curcuit-breaker.

#### Gate Seal behaviour (updated)

If, at any moment in time, two predicates become true simultaneuosly:

1. any DAO-managed contract functionality is paused by a Gate Seal;
2. the DAO execution is blocked by the DG mechanism (i.e. the global governance state is Veto Signalling, Veto Signalling Deactivation, or Rage Quit),

then the Gate Seal-induced pause is prolonged until the DAO execution is unblocked by the DG, i.e. until the global governance state becomes Normal or Veto Cooldown. 

Otherwise, the Gate Seal-induced pause lasts for a pre-defined fixed duration.

#### Tiebreaker Committee

Given the updated Gate Seal behaviour, the system allows for reaching a deadlock state: if the protocol withdrawals functionality gets paused by the Gate Seal committee while the governance state is Rage Quit, or if it gets paused before and remains paused until the Rage Quit starts, then withdrawals should remain paused until the Rage Quit state is exited and the DAO execution is unblocked. But the Rage Quit state lasts until all staked ETH participating in the rage quit is withdrawn to ETH, which cannot happen while withdrawals are paused.

Apart from the Gate Seal being activated, withdrawals can become dysfunctional due to a bug in the protocol code. If this happens while the Rage Quit state is active, it would also trigger the deadlock since a DAO proposal fixing the bug cannot be executed until the Rage Quit state is exited.

To resolve the potential deadlock, the mechanism contains a third-party arbiter **Tiebreaker Committee** elected by the DAO. The committee gains its power only under the specific conditions of the deadlock (see below), and the power is limited by bypassing the DG dynamic timelock for pending proposals approved by the DAO.

Specifically, the Tiebreaker committee can execute any pending proposal submitted by the DAO to DG, subject to a timelock of `TiebreakerExecutionTimelock` days, iff any of the following two conditions is true:

* **Tiebreaker Condition A**: (governance state is Rage Quit) AND (protocol withdrawals are paused by a Gate Seal).
* **Tiebreaker Condition B**: (governance state is Rage Quit) AND (last time governance exited Normal or Veto Cooldown state was more than `TiebreakerActivationTimeout` days ago).

The Tiebreaker committee should be composed of multiple sub-committees covering different interest groups within the Ethereum comminuty (e.g. largest DAOs, EF, L2s, node operators, OGs) and should require approval from a supermajority of sub-committees in order to execute a pending proposal. The approval by each sub-committee should require the majority support within the sub-committee. No sub-committee should contain more than $1/4$ of the members that are also members of the withdrawals Gate Seal committee.

The composition of the Tiebreaker committee should be set by a DAO vote (subject to DG) and reviewed at least once a year.

```env
# Proposed values, to be modeled and refined
TiebreakerExecutionTimelock = 1 month
TieBreakerActivationTimeout = 1 year
```

## Dual governance scope

Dual governance should cover any DAO proposal that could potentially affect the protocol users, including:

* Upgrading, adding, and removing any protocol code.
* Changing the global protocol parameters and safety limits.
* Changing the parameters of:
  * The withdrawal queue.
  * The staking router, including addition and removal of staking modules.
  * Staking modules, including addition and removal of node operators within the curated staking module.
* Adding, removing, and replacing the oracle committee members.
* Adding, removing, and replacing the deposit security committee members.

Importantly, any change to the parameters of the dual governance contracts (including managing the tiebreaker committee structure) should be also in the scope of dual governance.

Dual governance should not cover:

* Emergency actions triggered by curcuit breaker committees and contracts, including activation of any Gate Seal. These actions must be limited in scope and time and must be unable to change any protocol code.
* DAO decisions related to spending and managing the DAO treasury.


## Possible next iterations on governance and risk minimization

### More ways to activate veto

Assign veto power to node operators and/or client teams. For example, supermajority of node operators having the same power in activating the extended timelock and rage quit as $N\%$ of stETH (say, 10% so that only 5% of stETH is enough to trigger a rage quit).

### Extended timelocks on critical DAO decisions

Currently, all DAO decisions have the same static timelock.

Split all decision types into 2-3 groups by the potential impact on stakers and the network and assign different static timelock values to each of these groups, in addition to the dual governance user-activated dynamic timelock.

### Modular ossification

Modularize the protocol code so that stETH minting, transfers and withdrawals are detached ehough from the rest of the code to be either ossified or their upgrades heavily restricted.

Detaching the withdrawal flow may allow to significantly improve the Rage Quit state by only blocking the upgrades of parts of the code that can affect withdrawals processing. If these parts are formally verified on the bytecode level, this may allow to deprecate the Tiebreaker Committee and achieve better autonomy of the governance.

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

Any flavor of WC forking will most probably require the consensus layer support of WC rotation between WCs of the same type.

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

## Changelog

### 2024-03-22 (this document)

* Made the Veto Signalling Deactivation state duration dependent on the last time a proposal was submitted during Veto Signalling.

    > An example attack:
    > 1. A malicious DAO actor locks `SecondSealRageQuitSupport` tokens in the veto signalling escrow, triggering the transition to Veto Signalling.
    > 2. Waits until the `VetoSignallingMaxDuration` minus one block passes, submits a malicious proposal, and withdraws their tokens. This triggers the entrance of the Deactivation state.
    > 3. Now, honest stakers have only the `VetoSignallingDeactivationDuration` to enter the signalling escrow with no less than the `SecondSealRageQuitSupport` tokens and trigger rage quit. Otherwise, the Veto Cooldown state gets activated and the malicious proposal becomes executable.

* Added the Tiebreaker execution timelock.
* Renamed parameters and changed some terms for clarity.

### [2024-02-09](https://hackmd.io/@skozin/SyAR-PbxC)

* Removed the Rage Quit Accumulation state since it allowed a sophisticated actor to bypass locking (w)stETH in the escrow while still blocking the DAO execution (which, in turn, significantly reduced the cost of the "constant veto" DoS attack on the governance).
* Added details on veto signalling and rage quit escrows.
* Changed the post-rage quit ETH withdrawal timelock to be dynamic instead of static to further increase the cost of the "constant veto" DoS attack while keeping the default timelock adequate.

### [2023-12-05](https://hackmd.io/@skozin/r1U0T8-eC)

* Removed the stETH balance snapshotting mechanism since the Tiebreaker Committee already allows recovering from an infinite stETH mint vulnerability.
* Added support for using withdrawal NFTs in the veto escrow.

### [2023-10-23](https://hackmd.io/@skozin/SJdSE51Ep)

A major re-work of the DG mechanism (previous version [here](https://hackmd.io/@lido/BJKmFkM-i)).

* Replaced the global settlement mechanism with the rage quit mechanism (i.e. local settlement, a protected exit from the protocol). Removed states: Global Settlement; added states: Rage Quit Accumulation, Rage Quit.
* Removed the veto lift voting mechanism.
* Re-worked the DG activation and negotiation mechanism, replacing Veto Voting and Veto Negotiation states with the Veto Signalling state.
* Added the Veto Cooldown state.
* Added the $KillAllPendingProposals$ DAO decision.
* Added stETH balance snapshotting mechanism.
* Specified inter-operation between the Gate Seal mechanism and DG.
* Added the Tiebreaker Committee.
