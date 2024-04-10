# Dual Governance specification

**A version from 2024-04-04.**

---

Dual Governance (DG) is a governance subsystem that sits between the Lido DAO, represented by various voting systems, and the protocol contracts it manages. It protects protocol users from hostile actions by the DAO by allowing to cooperate and block any in-scope governance decision until either the DAO cancels this decision or users' (w)stETH is completely withdrawn to ETH.

This document provides the system description on the code architecture level. A detailed description on the mechanism level can be found in the [Dual Governance mechanism design][mech design] document which should be considered an integral part of this specification.

[mech design]: https://hackmd.io/@skozin/rkD1eUzja

[mech design - tiebreaker]: https://hackmd.io/@skozin/rkD1eUzja#Tiebreaker-Committee


## System overview

![image](https://hackmd.io/_uploads/Byivryiy0.png)

The system is composed of the following main contracts:

* [`DualGovernance.sol`](#Contract-DualGovernancesol) is a singleton that provides an interface for submitting governance proposals and scheduling their execution, as well as managing the list of supported proposers (DAO voting systems). Implements a state machine tracking the current global governance state which, in turn, determines whether proposal submission and execution is currently allowed.
* [`EmergencyProtectedTimelock.sol`](#Contract-EmergencyProtectedTimelocksol) is a singleton that stores submitted proposals and provides an interface for their execution. In addition, it implements an optional temporary protection from a zero-day vulnerability in the dual governance contracts following the initial deployment or upgrade of the system. The protection is implemented as a timelock on proposal execution combined with two emergency committees that have the right to cooperate and disable the dual governance.
* [`Executor.sol`](#Contract-Executorsol) contract instances make calls resulting from governance proposals' execution. Every protocol permission or role protected by the DG, as well as the permission to manage this role/permission, should be assigned exclusively to one of the instances of this contract (in contrast with being assigned directly to a DAO voting system).
* [`Escrow.sol`](#Contract-Escrowsol) is a contract that can hold stETH, wstETH, withdrawal NFTs, and plain ETH. It can exist in two states, each serving a different purpose: either an oracle for users' opposition to DAO proposals or an immutable and ungoverned accumulator for the ETH withdrawn as a result of the [rage quit](#Rage-quit).
* [`GateSealBreaker.sol`](#Contract-GateSealBreakersol) is a singleton that allows anyone to unpause the protocol contracts that were put into an emergency pause by the [GateSeal emergency protection mechanism](https://github.com/lidofinance/gate-seals), given that the minimum pause duration has passed and that the DAO execution is not currently blocked by the DG system.


## Proposal flow

The system supports multiple DAO voting systems, represented in the dual governance as proposers. A **proposer** is an address that has the right to submit sets of EVM calls (**proposals**) to be made by a dual governance's **executor contract**. Each proposer has a single associated executor, though multiple proposers can share the same executor, so the system supports multiple executors and the relation between proposers and executors is many-to-one.

![image](https://hackmd.io/_uploads/SJ2Jn_u0p.png)

The general proposal flow is the following:

1. A proposer submits a proposal, i.e. a set of EVM calls (represented by an array of [`ExecutorCall`](#Struct-ExecutorCall) structs) to be issued by the proposer's associated [executor contract](#Contract-Executorsol), by calling the [`DualGovernance.submitProposal`](#Function-DualGovernancesubmitProposal) function.
2. This starts a [dynamic timelock period](#Dynamic-timelock) that allows stakers to oppose the DAO, potentially leaving the protocol before the timelock elapses.
3. By the end of the dynamic timelock period, the proposal is either canceled by the DAO or executable.
    * If it's canceled, it cannot be scheduled for execution. However, any proposer is free to submit a new proposal with the same set of calls.
    * Otherwise, anyone can schedule the proposal for execution by calling the [`DualGovernance.scheduleProposal`](#Function-DualGovernancescheduleProposal) function, with the execution flow that follows being dependent on the [deployment mode](#Proposal-execution-and-deployment-modes).
4. The proposal's execution results in the proposal's EVM calls being issued by the executor contract associated with the proposer.


### Dynamic timelock

Each submitted proposal requires a minimum timelock before it can be scheduled for execution.

At any time, including while a proposal's timelock is lasting, stakers can signal their opposition to the DAO by locking their (w)stETH or withdrawal NFTs (wNFTs) into the [signalling escrow contract](#Contract-Escrowsol). If the opposition exceeds some minimum threshold, the [global governance state](#Governance-state) gets changed, blocking any DAO execution and thus effectively extending the timelock of all pending (i.e. submitted but not scheduled for execution) proposals.

![image](https://hackmd.io/_uploads/SyAKgF_Ra.png)

At any time, the DAO can cancel all pending proposals by calling the [`DualGovernance.cancelAllPendingProposals`](#Function-DualGovernancecancelAllPendingProposals) function.

By the time the dynamic timelock described above elapses, one of the following outcomes is possible:

* The DAO was not opposed by stakers (the **happy path** scenario).
* The DAO was opposed by stakers and canceled all pending proposals (the **two-sided de-escalation** scenario).
* The DAO was opposed by stakers and didn't cancel pending proposals, forcing the stakers to leave via the rage quit process, or canceled the proposals but some stakers still left (the **rage quit** scenario).
* The DAO was opposed by stakers and didn't cancel pending proposals but the total stake opposing the DAO was too small to trigger the rage quit (the **failed escalation** scenario).


### Proposal execution and deployment modes

The proposal execution flow comes after the dynamic timelock elapses and the proposal is scheduled for execution. The system can function in two deployment modes which affect the flow.

![image](https://hackmd.io/_uploads/SkknCeoJR.png)

#### Regular deployment mode

In the regular deployment mode, the emergency protection delay is set to zero and all calls from scheduled proposals are immediately executable by anyone via calling the [`EmergencyProtectedTimelock.execute`](#Function-EmergencyProtectedTimelockexecute) function.

#### Protected deployment mode

The protected deployment mode is a temporary mode designed to be active during an initial period after the deployment or upgrade of the DG contracts. In this mode, scheduled proposals cannot be executed immediately; instead, before calling [`EmergencyProtectedTimelock.execute`](#Funtion-EmergencyProtectedTimelockexecute), one has to wait until an **emergency protection timelock** elapses since the proposal scheduling time.

![image](https://hackmd.io/_uploads/rJGZZQh1A.png)

In this mode, an **emergency activation committee** has the one-off and time-limited right to activate an adversarial **emergency mode** if they see a scheduled proposal that was created or altered due to a vulnerability in the DG contracts or if governance execution is prevented by such a vulnerability. Once the emergency mode is activated, the emergency activation committee is disabled, i.e. loses the ability to activate the emergency mode again. If the emergency activation committee doesn't activate the emergency mode within the duration of the **emergency protection duration** since the committee was configured by the DAO, it gets automatically disabled as well.

The emergency mode lasts up to the **emergency mode max duration** counting from the moment of its activation. While it's active, 1) only the **emergency execution committee** has the right to execute scheduled proposals, and 2) the same committee has the one-off right to **disable the DG subsystem**, i.e. disconnect executor contracts from the DG contracts and reconnect them to the Lido DAO Voting/Agent contract. The latter also disables the emergency mode and the emergency execution committee, so any proposal can be executed by the DAO without cooperation from any other actors.

If the emergency execution committee doesn't disable the DG until the emergency mode max duration elapses, anyone gets the right to deactivate the emergency mode, switching the system back to the protected mode and disabling the emergency committee.

:::info
Note: the protected deployment mode and emergency mode are only designed to protect from a vulnerability in the DG contracts and assume the honest and operational DAO. The system is not designed to handle a situation when there's a vulnerability in the DG contracts AND the DAO is captured/malicious or otherwise dysfunctional.
:::


## Governance state

The DG system implements a state machine tracking the **global governance state** defining which governance actions are currently possible. The state is global since it affects all non-executed proposals and all system actors.

The state machine is specified in the [Dual Governance mechanism design][mech design] document. The possible states are:

* `Normal` allows proposal submission and scheduling for execution.
* `VetoSignalling` only allows proposal submission.
    * `VetoSignallingDeactivation` sub-state (doesn't deactivate the parent state upon entry) doesn't allow proposal submission or scheduling for execution.
* `VetoCooldown` only allows scheduling already submitted proposals for execution.
* `RageQuit` only allows proposal submission.

![image](https://hackmd.io/_uploads/SkDcy5Y1C.png)

Possible state transitions:

* `Normal` → `VetoSignalling`
* `VetoSignalling` → `RageQuit`
* `VetoSignallingDeactivation` sub-state entry and exit (while the parent `VetoSignalling` state is active)
* `VetoSignallingDeactivation` → `VetoCooldown`
* `VetoCooldown` → `Normal`
* `VetoCooldown` → `VetoSignalling`
* `RageQuit` → `VetoCooldown`
* `RageQuit` → `VetoSignalling`

These transitions are enabled by three processes (see the [mechanism design document][mech design] for more details):

1. **Rage quit support** changing due to stakers locking and unlocking their tokens into/out of the veto signalling escrow or stETH total supply changing;
2. Protocol withdrawals processing (in the `RageQuit` state);
3. Time passing.

![image](https://hackmd.io/_uploads/BJSm_cKyC.png)


## Rage quit

Rage quit is a global process of withdrawing stETH and wstETH locked in the signalling escrow and waiting until all these withdrawals, as well as any withdrawals represented by withdrawal NFTs that were locked into the signalling escrow prior to the process started, are finished.

![image](https://hackmd.io/_uploads/rk9vD_dC6.png)

In the [governance state machine](#Governance-state), the rage quit process is represented by the `RageQuit` global state. While this state is active, no proposal can be scheduled for execution. Thus, rage quit contributes to dynamic timelocks of all pending proposals.

At any time, only one instance of the rage quit process can be active.

From the stakers' point of view, opposition to the DAO and the rage quit process can be described by the following diagram:

![image](https://hackmd.io/_uploads/SylRb5K1C.png)


## Tiebreaker committee

The mechanism design allows for a deadlock where the system is stuck in the `RageQuit` state while protocol withdrawals are paused or dysfunctional and require a DAO vote to resume, and includes a third-party arbiter Tiebreaker committee for resolving it.

The committee gains the power to bypass the DG dynamic timelock and execute pending proposals under the specific conditions of the deadlock. The detailed Tiebreaker mechanism design can be found in the [Dual Governance mechanism design overview][mech design - tiebreaker] document.

The Tiebreaker committee is represented in the system by its address which can be configured via the admin executor calling the [`DualGovernance.setTiebreakerCommittee`](#Function-DualGovernancesetTiebreakerCommittee) function.

While the deadlock conditions are met, the tiebreaker committee address is allowed to approve execution of any pending proposal by calling [`DualGovernance.tiebreakerApproveProposal`](#Function-DualGovernancetiebreakerApproveProposal) so that its execution can be scheduled after the tiebreaker execution timelock passes by calling [`DualGovernance.tiebreakerScheduleProposal`](#Function-DualGovernancetiebreakerScheduleProposal).


## Administrative actions

The dual governance system supports a set of administrative actions, including:

* Changing the configuration options.
* [Upgrading the system's code](#Upgrade-flow-description).
* Managing the [deployment mode](#Proposal-execution-and-deployment-modes): configuring or disabling the emergency protection delay, setting the emergency committee addresses and lifetime.
* Setting the [Tiebreaker committee](#Tiebreaker-committee) address.

Each of these actions can only be performed by a designated **admin executor** contract (set by a configuration option), meaning that:

1. It has to be proposed by one of the proposers associated with this executor. Such proposers are called **admin proposers**.
2. It has to go through the dual governance execution flow with stakers having the power to object.


## Common types

### Struct: ExecutorCall

```solidity
struct ExecutorCall {
    address target;
    uint96 value;
    bytes payload;
}
```

Encodes an EVM call from an executor contract to the `target` address with the specified `value` and the calldata being set to `payload`.


## Contract: DualGovernance.sol

The main entry point to the dual governance system. 

* Provides an interface for submitting and cancelling governance proposals and implements a dynamic timelock on scheduling their execution.
* Manages the list of supported proposers (DAO voting systems).
* Implements a state machine tracking the current [global governance state](#Governance-state) which, in turn, determines whether proposal submission and execution is currently allowed.
* Deploys and tracks the [`Escrow`](#Contract-Escrowsol) contract instances. Tracks the current signalling escrow.

This contract is a singleton, meaning that any DG deployment includes exectly one instance of this contract.


### Enum: DualGovernance.State

```solidity
enum State {
    Normal,
    VetoSignalling,
    VetoSignallingDeactivation,
    VetoCooldown,
    RageQuit
}
```

Encodes the current global [governance state](#Governance-state), affecting the set of actions allowed for each of the system's actors.


### Function: DualGovernance.submitProposal

```solidity
function submitProposal(ExecutorCall[] calls)
  returns (uint256 proposalId)
```

Instructs the [`EmergencyProtectedTimelock`](#Contract-EmergencyProtectedTimelocksol) singleton instance to register a new governance proposal composed of one or more EVM `calls` to be made by an executor contract currently associated with the proposer address calling this function. Starts a dynamic timelock on [scheduling the proposal](#Function-DualGovernancescheduleProposal) for execution.

See: [`EmergencyProtectedTimelock.submit`](#Function-EmergencyProtectedTimelocksubmit).

#### Returns

The id of the successfully registered proposal.

#### Preconditions

* The calling address MUST be [registered as a proposer](#Function-DualGovernanceregisterProposer).
* The current governance state MUST be either of: `Normal`, `VetoSignalling`, `RageQuit`.

Triggers a transition of the current governance state (if one is possible) before checking the preconditions.


### Function: DualGovernance.scheduleProposal

```solidity
function scheduleProposal(uint256 proposalId)
```

Instructs the [`EmergencyProtectedTimelock`](#Contract-EmergencyProtectedTimelocksol) singleton instance to schedule the proposal with id `proposalId` for execution.

#### Preconditions

* The proposal with the given id MUST be already submitted.
* The proposal MUST NOT be scheduled.
* The proposal's dynamic timelock MUST have elapsed.
* The proposal MUST NOT be canceled.
* The current governance state MUST be either `Normal` or `VetoCooldown`.

Triggers a transition of the current governance state (if one is possible) before checking the preconditions.

### Function: DualGovernance.tiebreakerApproveProposal

```solidity
function tiebreakerApproveProposal(uint256 proposalId)
```

Marks the proposal with id `proposalId` as approved by the [Tiebreaker committee](#Tiebreaker-committee), given that the DG system is in a deadlock.

#### Preconditions

* MUST be called by the [Tiebreaker committee address](#Function-DualGovernancesetTiebreakerCommittee).
* Either the Tiebreaker Condition A or the Tiebreaker Condition B MUST be met (see the [mechanism design document][mech design - tiebreaker]).
* The proposal MUST be already submitted.
* The proposal MUST NOT be canceled.
* The proposal with the specified id MUST NOT be already approved by the Tiebreaker committee.

Triggers a transition of the current governance state (if one is possible) before checking the preconditions.


### Function: DualGovernance.tiebreakerScheduleProposal

```solidity
function tiebreakerScheduleProposal(uint256 proposalId)
```

Instructs the [`EmergencyProtectedTimelock`](#Contract-EmergencyProtectedTimelocksol) singleton instance to schedule the proposal with the id `proposalId` for execution, bypassing the proposal dynamic timelock and given that the proposal was previously approved by the [Tiebreaker committee](#Tiebreaker-committee) and that the tiebreaker execution timelock has elapsed.

#### Preconditions

* Either the Tiebreaker Condition A or the Tiebreaker Condition B MUST be met (see the [mechanism design document][mech design - tiebreaker]).
* The proposal MUST be already submitted.
* The proposal MUST NOT be canceled.
* The proposal with the specified id MUST be approved by the Tiebreaker committee.
* The current block timestamp MUST be at least `TIEBREAKER_EXECUTION_TIMELOCK` seconds greater than the timestamp of the block in which the proposal was approved by the Tiebreaker committee.

Triggers a transition of the current governance state (if one is possible) before checking the preconditions.


### Function: DualGovernance.cancelAllPendingProposals

```solidity
function cancelAllPendingProposals()
```

Cancels all currently submitted and non-executed proposals. If a proposal was submitted but not scheduled, it becomes unschedulable. If a proposal was scheduled, it becomes unexecutable.

Triggers a transition of the current governance state, if one is possible.

#### Preconditions

* MUST be called by an [admin proposer](#Administrative-actions).


### Function: DualGovernance.registerProposer

```solidity
function registerProposer(address proposer, address executor)
```

Registers the `proposer` address in the system as a valid proposer and associates it with the `executor` contract address (which is expected to be an instance of [`Executor.sol`](#Contract-Executorsol)) as an executor.

#### Preconditions

* MUST be called by the admin executor contract (see `Config.sol`).
* The `proposer` address MUST NOT be already registered in the system.
* The `executor` instance SHOULD be owned by the [`EmergencyProtectedTimelock`](#Contract-EmergencyProtectedTimelocksol) singleton instance.


### Function: DualGovernance.unregisterProposer

```solidity
function unregisterProposer(address proposer)
```

Removes the registered `proposer` address from the list of valid proposers and dissociates it with the executor contract address.

#### Preconditions

* MUST be called by the admin executor contract.
* The `proposer` address MUST be registered in the system as proposer.


### Function: DualGovernance.setTiebreakerCommittee

```solidity
function setTiebreakerCommittee(address newTiebreaker)
```

Updates the address of the [Tiebreaker committee](#Tiebreaker-committee).

#### Preconditions

* MUST be called by the admin executor contract.


### Function: DualGovernance.activateNextState

```solidity
function activateNextState()
```

Triggers a transition of the [global governance state](#Governance-state), if one is possible; does nothing otherwise.


## Contract: Executor.sol

Issues calls resulting from governance proposals' execution. Every protocol permission or role protected by the DG, as well as the permission to manage this role/permission, should be assigned exclusively to the instances of this contract.

The system supports multiple instances of this contract, but all instances SHOULD be owned by the [`EmergencyProtectedTimelock`](#Contract-EmergencyProtectedTimelocksol) singleton instance.

### Function: execute

```solidity
function execute(address target, uint256 value, bytes payload)
  payable returns (bytes result)
```

Issues a EVM call to the `target` address with the `payload` calldata, optionally sending `value` wei ETH. 

Reverts if the call was unsuccessful.

#### Returns

The result of the call.

#### Preconditions

* MUST be called by the contract owner (which SHOULD be the [`EmergencyProtectedTimelock`](#Contract-EmergencyProtectedTimelocksol) singleton instance).


## Contract: Escrow.sol

The `Escrow` contract serves as an accumulator of users' (w)stETH, withdrawal NFTs, and ETH. It has two internal states and serves a different purpose depending on its state:

* The initial state is the `SignallingEscrow` state.  In this state, the contract serves as an oracle for users' opposition to DAO proposals. It allows users to lock and unlock (unlocking is permitted only for the caller after the `SignallingEscrowMinLockTime` duration has passed since their last funds locking operation) stETH, wstETH, and withdrawal NFTs, potentially changing the global governance state. The `SignallingEscrowMinLockTime` duration, measured in hours, safeguards against manipulating the dual governance state through instant lock/unlock actions within the `Escrow` contract instance.
* The final state is the `RageQuitEscrow` state. In this state, the contract serves as an immutable and ungoverned accumulator for the ETH withdrawn as a result of the [rage quit](#Rage-quit) and enforces a timelock on reclaiming this ETH by users.

The `DualGovernance` contract tracks the current signalling escrow contract using the `DualGovernance.signallingEscrow` pointer. Upon the initial deployment of the system, an instance of `Escrow` is deployed in the `SignallingEscrow` state by the `DualGovernance` contract and the `DualGovernance.signallingEscrow` pointer is set to this contract.

Each time the governance enters the global `RageQuit` state, two things happen simultaneously:

1. The `Escrow` instance currently stored in the `DualGovernance.signallingEscrow` pointer changes its state from `SignallingEscrow` to `RageQuitEscrow`. This is the only possible (and thus irreversible) state transition.
2. The `DualGovernance` contract deploys a new instance of `Escrow` in the `SignallingEscrow` state and resets the `DualGovernance.signallingEscrow` pointer to this newly-deployed contract.

At any point in time, there can be only one instance of the contract in the `SignallingEscrow` state (so the contract in this state is a singleton) but multiple instances of the contract in the `RageQuitEscrow` state.

After the `Escrow` instance transitions into the `RageQuitEscrow` state, all locked stETH and wstETH tokens are meant to be converted into withdrawal NFTs using the permissionless `Escrow.requestNextWithdrawalsBatch()` function.

Once all funds locked in the `Escrow` instance are converted into withdrawal NFTs, finalized, and claimed, the main rage quit phase concludes, and the `RageQuitExtensionDelay` period begins.

The purpose of the `RageQuitExtensionDelay` phase is to provide sufficient time to participants who locked withdrawal NFTs to claim them before Lido DAO's proposal execution is unblocked. As soon as a withdrawal NFT is claimed, the user's ETH is no longer affected by any code controlled by the DAO.

When the `RageQuitExtensionDelay` period elapses, the `DualGovernance.activateNextState()` function exits the `RageQuit` state and initiates the `RageQuitEthClaimTimelock`. Throughout this timelock, tokens remain locked within the `Escrow` instance and are inaccessible for withdrawal. Once the timelock expires, participants in the rage quit process can retrieve their ETH by withdrawing it from the `Escrow` instance.

The duration of the `RageQuitEthClaimTimelock` is dynamic and varies based on the number of "continuous" rage quits. A pair of rage quits is considered continuous when `DualGovernance` has not transitioned to the `Normal` or `VetoCooldown` state between them.

### Function: Escrow.lockStETH

```solidity!
function lockStETH(uint256 amount)
```

Transfers the specified `amount` of stETH from the caller's (i.e., `msg.sender`) account into the `SignallingEscrow` instance of the `Escrow` contract.

The total rage quit support is updated proportionally to the number of shares corresponding to the locked stETH (see the `Escrow.getRageQuitSupport()` function for the details). For the correct rage quit support calculation, the function updates the number of locked stETH shares in the protocol as follows:

```solidity
uint256 amountInShares = stETH.getSharesByPooledEther(amount);

_vetoersLockedAssets[msg.sender].stETHShares += amountInShares;
_totalStEthSharesLocked += amountInShares;
```

The rage quit support will be dynamically updated to reflect changes in the stETH balance due to protocol rewards or validators slashing.

Finally, calls the `DualGovernance.activateNextState()` function. This action may transit the `Escrow` instance from the `SignallingEscrow` state into the `RageQuitEscrow` state.

#### Preconditions

- The `Escrow` instance MUST be in the `SignallingEscrow` state.
- The caller MUST have an allowance set on the stETH token for the `Escrow` instance equal to or greater than the locked `amount`.
- The locked `amount` MUST NOT exceed the caller's stETH balance.

### Function: Escrow.unlockStETH

```solidity
function unlockStETH()
```
  
Allows the caller (i.e. `msg.sender`) to unlock the previously locked stETH in the `SignallingEscrow` instance of the `Escrow` contract. The locked stETH balance may change due to protocol rewards or validators slashing, potentially altering the original locked amount. The total unlocked stETH equals the sum of all previously locked stETH by the caller, accounting for any changes during the locking period.

For the correct rage quit support calculation, the function updates the number of locked stETH shares in the protocol as follows:

```solidity
_totalStEthSharesLocked -= _vetoersLockedAssets[msg.sender].stETHShares;
_vetoersLockedAssets[msg.sender].stETHShares = 0;
```

Additionally, the function triggers the `DualGovernance.activateNextState()` function at the beginning and end of the execution.

#### Preconditions

- The `Escrow` instance MUST be in the `SignallingEscrow` state.
- The caller MUST have a non-zero amount of previously locked stETH in the `Escrow` instance using the `Escrow.lockStETH` function.
- At least the duration of the `SignallingEscrowMinLockTime` MUST have passed since the caller last invoked any of the methods `Escrow.lockStETH`, `Escrow.lockWstETH`, or `Escrow.lockUnstETH`.

### Function: Escrow.lockWstETH

```solidity
function lockWstETH(uint256 amount)
```

Transfers the specified `amount` of wstETH from the caller's (i.e., `msg.sender`) account into the `SignallingEscrow` instance of the `Escrow` contract.

The total rage quit support is updated proportionally to the  `amount` of locked wstETH (see the `Escrow.getRageQuitSupport()` function for the details). For the correct rage quit support calculation, the function updates the number of locked wstETH in the protocol as follows:

```solidity
_vetoersLockedAssets[msg.sender].wstETHShares += amount;
_totalStEthSharesLocked += amount;
```

Finally, calls the `DualGovernance.activateNextState()` function. This action may transit the `Escrow` instance from the `SignallingEscrow` state into the `RageQuitEscrow` state.

#### Preconditions

- The `Escrow` instance MUST be in the `SignallingEscrow` state.
- The caller MUST have an allowance set on the wstETH token for the `Escrow` instance equal to or greater than the locked `amount`.
- The locked `amount` MUST NOT exceed the caller's wstETH balance.

### Function: Escrow.unlockWstETH

```solidity
function unlockWstETH()
```

Allows the caller (i.e. `msg.sender`) to unlock previously locked wstETH from the `SignallingEscrow` instance of the `Escrow` contract. The total unlocked wstETH equals the sum of all previously locked wstETH by the caller.

For the correct rage quit support calculation, the function updates the number of locked wstETH shares in the protocol as follows:

```solidity
_totalStEthSharesLocked -= _vetoersLockedAssets[msg.sender].wstETHShares;
_vetoersLockedAssets[msg.sender].wstETHShares = 0;
```

Additionally, the function triggers the `DualGovernance.activateNextState()` function at the beginning and end of the execution.

#### Preconditions

- The `Escrow` instance MUST be in the `SignallingEscrow` state.
- The caller MUST have a non-zero amount of previously locked wstETH in the `Escrow` instance using the `Escrow.lockWstETH` function.
- At least the duration of the `SignallingEscrowMinLockTime` MUST have passed since the caller last invoked any of the methods `Escrow.lockStETH`, `Escrow.lockWstETH`, or `Escrow.lockUnstETH`.


### Function: Escrow.lockUnstETH

```solidity
function lockUnstETH(uint256[] unstETHIds)
```

Transfers the WIthdrawal NFTs with ids contained in the `unstETHIds` from the caller's (i.e. `msg.sender`) account into the `SignallingEscrow` instance of the `Escrow` contract. 

To correctly calculate the rage quit support (see the `Escrow.getRageQuitSupport()` function for the details), updates the number of locked Withdrawal NFT shares in the protocol for each withdrawal NFT in the `unstETHIds`,  as follows:

```solidity
uint256 amountOfShares = WithdrawalRequest[id].amountOfShares;

_vetoersLockedAssets[msg.sender].withdrawalNFTShares += amountOfShares;
_totalWithdrawlNFTSharesLocked += amountOfShares;
```

Finally, calls the `DualGovernance.activateNextState()` function. This action may transition the `Escrow` instance from the `SignallingEscrow` state into the `RageQuitEscrow` state.

#### Preconditions

- The `Escrow` instance MUST be in the `SignallingEscrow` state.
- The caller MUST be the owner of all withdrawal NFTs with the given ids.
- The caller MUST grant permission to the `SignallingEscrow` instance to transfer tokens with the given ids (`approve()` or `setApprovalForAll()`).
- The passed ids MUST NOT contain the finalized or claimed Withdrawal NFTs.
- The passed ids MUST NOT contain duplicates.

### Function: Escrow.unlockUnstETH

```solidity
function unlockUnstETH(uint256[] unstETHIds)
```

Allows the caller (i.e. `msg.sender`) to unlock a set of previously locked Withdrawal NFTs with ids `unstETHIds` from the `SignallingEscrow` instance of the `Escrow` contract.

To correctly calculate the rage quit support (see the `Escrow.getRageQuitSupport()` function for details), updates the number of locked Withdrawal NFT shares in the protocol for each withdrawal NFT in the `unstETHIds`, as follows:

- If the Withdrawal NFT was marked as finalized (see the `Escrow.markUnstETHFinalized()` function for details):

```solidity
uint256 amountOfShares = WithdrawalRequest[id].amountOfShares;
uint256 claimableAmount = _getClaimableEther(id);

_totalWithdrawlNFTSharesLocked -= amountOfShares;
_totalFinalizedWithdrawlNFTSharesLocked -= amountOfShares;
_totalFinalizedWithdrawlNFTAmountLocked -= claimableAmount;

_vetoersLockedAssets[msg.sender].withdrawalNFTShares -= amountOfShares;
_vetoersLockedAssets[msg.sender].finalizedWithdrawalNFTShares -= amountOfShares;
_vetoersLockedAssets[msg.sender].finalizedWithdrawalNFTAmount -= claimableAmount;
```

- if the Withdrawal NFT wasn't marked as finalized:

```solidity
uint256 amountOfShares = WithdrawalRequest[id].amountOfShares;

_totalWithdrawlNFTSharesLocked -= amountOfShares;
_vetoersLockedAssets[msg.sender].withdrawalNFTShares -= amountOfShares;
```

Additionally, the function triggers the `DualGovernance.activateNextState()` function at the beginning and end of the execution.

#### Preconditions

- The `Escrow` instance MUST be in the `SignallingEscrow` state.
- Each provided Withdrawal NFT MUST have been previously locked by the caller.
- At least the duration of the `SignallingEscrowMinLockTime` MUST have passed since the caller last invoked any of the methods `Escrow.lockStETH`, `Escrow.lockWstETH`, or `Escrow.lockUnstETH`.

### Function Escrow.markUnstETHFinalized

```solidity
function markUnstETHFinalized(uint256[] unstETHIds, uint256[] hints)
```

Marks the provided Withdrawal NFTs with ids `unstETHIds` as finalized to accurately calculate their rage quit support.

The finalization of the Withdrawal NFT leads to the following events:

- The value of the Withdrawal NFT is no longer affected by stETH token rebases.
- The total supply of stETH is adjusted based on the value of the finalized Withdrawal NFT.

As both of these events affect the rage quit support value, this function updates the number of finalized Withdrawal NFTs for the correct rage quit support accounting.

For each Withdrawal NFT in the `unstETHIds`:

```solidity
uint256 claimableAmount = _getClaimableEther(id);
uint256 amountOfShares = WithdrawalRequest[id].amountOfShares;

_totalFinalizedWithdrawlNFTSharesLocked += amountOfShares;
_totalFinalizedWithdrawlNFTAmountLocked += claimableAmount;

_vetoersLockedAssets[msg.sender].finalizedWithdrawalNFTShares += amountOfShares;
_vetoersLockedAssets[msg.sender].finalizedWithdrawalNFTAmount += claimableAmount;
```

Withdrawal NFTs belonging to any of the following categories are excluded from the rage quit support update:

- Claimed or unfinalized Withdrawal NFTs
- Withdrawal NFTs already marked as finalized
- Withdrawal NFTs not locked in the `Escrow` instance

#### Preconditions

- The `Escrow` instance MUST be in the `SignallingEscrow` state.

### Function Escrow.getRageQuitSupport()

```solidity
function getRageQuitSupport() view returns (uint256)
```

Calculates and returns the total rage quit support as a percentage of the stETH total supply locked in the instance of the `Escrow` contract. It considers contributions from stETH, wstETH, and non-finalized Withdrawal NFTs while adjusting for the impact of locked finalized Withdrawal NFTs.

The returned value represents the total rage quit support expressed as a percentage with a precision of 16 decimals. It is computed using the following formula:

```solidity
uint256 rebaseableAmount = stETH.getPooledEthByShares(
	_totalStEthSharesLocked + 
	_totalWstEthSharesLocked +
	_totalWithdrawalNFTSharesLocked -
	_totalFinalizedWithdrawalNFTSharesLocked
);

return 10 ** 18 * (
    rebaseableAmount + _totalFinalizedWithdrawalNFTAmountLocked
) / (
    stETH.totalSupply() + _totalFinalizedWithdrawalNFTAmountLocked
);
```

### Function Escrow.startRageQuit

```solidity
function startRageQuit()
```

Transits the `Escrow` instance from the `SignallingEscrow` state to the `RageQuitEscrow` state. Following this transition, locked funds become unwithdrawable and are accessible to users only as plain ETH after the completion of the full `RageQuit` process, including the `RageQuitExtensionDelay` and `RageQuitEthClaimTimelock` stages.

As the initial step of transitioning to the `RageQuitEscrow` state, all locked wstETH is converted into stETH, and the maximum stETH allowance is granted to the `WithdrawalQueue` contract for the upcoming creation of Withdrawal NFTs.

#### Preconditions

- Method MUST be called by the `DualGovernance` contract.
- The `Escrow` instance MUST be in the `SignallingEscrow` state.

### Function Escrow.requestNextWithdrawalsBatch

```solidity
function requestNextWithdrawalsBatch(uint256 maxWithdrawalRequestsCount)
```

Transfers stETH held in the `RageQuitEscrow` instance into the `WithdrawalQueue`. The function may be invoked multiple times until all stETH is converted into Withdrawal NFTs. For each Withdrawal NFT, the owner is set to `Escrow` contract instance. Each call creates up to `maxWithdrawalRequestsCount` withdrawal requests, where each withdrawal request size equals `WithdrawalQueue.MAX_STETH_WITHDRAWAL_AMOUNT()`, except for potentially the last batch, which may have a smaller size.

Upon execution, the function updates the count of withdrawal requests generated by all invocations. When the remaining stETH balance on the contract falls below `WITHDRAWAL_QUEUE.MIN_STETH_WITHDRAWAL_AMOUNT()`, the generation of withdrawal batches is concluded, and subsequent function calls will revert.

#### Preconditions

- The `Escrow` instance MUST be in the `RageQuitEscrow` state.
- The `maxWithdrawalRequestsCount` MUST be greater than 0
- The generation of WithdrawalRequest batches MUST not be concluded

### Function Escrow.claimNextWithdrawalsBatch

```solidity
function claimNextWithdrawalsBatch(uint256[] withdrawalRequestIds, uint256[] hints)
```

Allows users to claim finalized Withdrawal NFTs generated by the `Escrow.requestNextWithdrawalsBatch()` function. 
Tracks the total amount of claimed ETH updating the `_totalClaimedEthAmount` variable. Upon claiming the last batch, the `RageQuitExtensionDelay` period commences.

#### Preconditions

- The `Escrow` instance MUST be in the `RageQuitEscrow` state.
- The `withdrawalRequestIds` array MUST contain only the ids of finalized but unclaimed withdrawal requests generated by the `Escrow.requestNextWithdrawalsBatch()` function.

### Function Escrow.claimUnstETH

```solidity
function claimUnstETH(uint256[] unstETHIds, uint256[] hints)
```

Allows users to claim the ETH associated with finalized Withdrawal NFTs with ids `unstETHIds` locked in the `Escrow` contract. Upon calling this function, the claimed ETH is transferred to the `Escrow` contract instance.

To safeguard the ETH associated with Withdrawal NFTs, this function should be invoked when the `Escrow` is in the `RageQuitEscrow` state and before the `RageQuitExtensionDelay` period ends. The ETH corresponding to unclaimed Withdrawal NFTs after this period ends would still be controlled by the code potentially afftected by pending and future DAO decisions.

#### Preconditions

- The `Escrow` instance MUST be in the `RageQuitEscrow` state.
- The provided `unstETHIds` MUST only contain finalized but unclaimed withdrawal requests with the owner set to `msg.sender`.

### Function Escrow.isRageQuitFinalized

```solidity
function isRageQuitFinalized() view returns (bool)
```

Returns whether the rage quit process has been finalized. The rage quit process is considered finalized when all the following conditions are met:
- The `Escrow` instance is in the `RageQuitEscrow` state.
- All withdrawal request batches have been claimed.
- The duration of the `RageQuitExtensionDelay` has elapsed.

### Function Escrow.withdrawStEthAsEth

```solidity
function withdrawStEthAsEth()
```

Allows the caller (i.e. `msg.sender`) to withdraw all stETH they have previouusly locked into `Escrow` contract instance (while it was in the `SignallingEscrow` state) as plain ETH, given that the `RageQuit` process is completed and that the `RageQuitEthClaimTimelock` has elapsed. Upon execution, the function transfers ETH to the caller's account and marks the corresponding stETH as withdrawn for the caller.

The amount of ETH sent to the caller is determined by the proportion of the user's stETH shares compared to the total amount of locked stETH and wstETH shares in the Escrow instance, calculated as follows:

```solidity
return _totalClaimedEthAmount * _vetoersLockedAssets[msg.sender].stETHShares
    / (_totalStEthSharesLocked + _totalWstEthSharesLocked);
```

#### Preconditions

- The `Escrow` instance MUST be in the `RageQuitEscrow` state.
- The rage quit process MUST be completed, including the expiration of the `RageQuitExtensionDelay` duration.
- The `RageQuitEthClaimTimelock` period MUST be elapsed after the expiration of the `RageQuitExtensionDelay` duration.
- The caller MUST have a non-zero amount of stETH to withdraw.
- The caller MUST NOT have previously withdrawn stETH.

### Function Escrow.withdrawWstEthAsEth

```solidity
function withdrawWstEthAsEth() external
```

Allows the caller (i.e. `msg.sender`) to withdraw all wstETH they have previouusly locked into `Escrow` contract instance (while it was in the `SignallingEscrow` state) as plain ETH, given that the `RageQuit` process is completed and that the `RageQuitEthClaimTimelock` has elapsed. Upon execution, the function transfers ETH to the caller's account and marks the corresponding wstETH as withdrawn for the caller.

The amount of ETH sent to the caller is determined by the proportion of the user's wstETH funds compared to the total amount of locked stETH and wstETH shares in the Escrow instance, calculated as follows:

```solidity
return _totalClaimedEthAmount * 
  _vetoersLockedAssets[msg.sender].wstETHShares /
  (_totalStEthSharesLocked + _totalWstEthSharesLocked);
```

#### Preconditions
- The `Escrow` instance MUST be in the `RageQuitEscrow` state.
- The rage quit process MUST be completed, including the expiration of the `RageQuitExtensionDelay` duration.
- The `RageQuitEthClaimTimelock` period MUST be elapsed after the expiration of the `RageQuitExtensionDelay` duration.
- The caller MUST have a non-zero amount of wstETH to withdraw.
- The caller MUST NOT have previously withdrawn wstETH.

### Function Escrow.withdrawUnstETHAsEth

```solidity
function withdrawUnstETHAsEth(uint256[] unstETHIds)
```

Allows the caller (i.e. `msg.sender`) to withdraw the claimed ETH from the Withdrawal NFTs with ids `unstETHIds` locked by the caller in the `Escrow` contract while the latter was in the `SignallingEscrow` state. Upon execution, all ETH previously claimed from the NFTs is transferred to the caller's account, and the NFTs are marked as withdrawn.

#### Preconditions

- The `Escrow` instance MUST be in the `RageQuitEscrow` state.
- The rage quit process MUST be completed, including the expiration of the `RageQuitExtensionDelay` duration.
- The `RageQuitEthClaimTimelock` period MUST be elapsed after the expiration of the `RageQuitExtensionDelay` duration.
- The caller MUST be set as the owner of the provided NFTs.
- Each Withdrawal NFT MUST have been claimed using the `Escrow.claimUnstETH()` function.
- Withdrawal NFTs must not have been withdrawn previously.


## Contract: EmergencyProtectedTimelock.sol

`EmergencyProtectedTimelock` is the singleton instance storing proposals approved by DAO voting systems and submitted to the Dual Governance. It allows for setting up time-bound **Emergency Activation Committee** and **Emergency Execution Committee**, acting as safeguards for the case of zero-day vulnerability in Dual Governance contracts.

For a proposal to be executed, the following steps have to be performed in order:

1. The proposal must be submitted using the `EmergencyProtectedTimelock.submit` function.
2. The configured post-submit timelock must elapse.
3. The proposal must be scheduled using the `EmergencyProtectedTimelock.schedule` function.
4. The configured emergency protection timelock must elapse (can be zero, see below).
5. The proposal must be executed using the `EmergencyProtectedTimelock.execute` function.

The contract only allows proposal submission and scheduling by the `governance` address. Normally, this address points to the [`DualGovernance`](#Contract-DualGovernancesol) singleton instance. Proposal execution is permissionless, unless Emergency Mode is activated.

If the Emergency Committees are set up and active, the governance proposal gets a separate emergency protection timelock between submitting and scheduling. This additional timelock is implemented in the `EmergencyProtectedTimelock` contract to protect from zero-day vulnerability in the logic of `DualGovenance.sol` and other core DG contracts. If the Emergency Committees aren't set, the proposal flow is the same, but the timelock duration is zero.

Emergency Activation Committee, while active, can enable the Emergency Mode. This mode prohibits anyone but the Emergency Execution Committee from executing proposals. It also allows the Emergency Execution Committee to reset the governance, effectively disabling the Dual Governance subsystem.

The governance reset entails the following steps:

1. Clearing both the Emergency Activation and Execution Committees from the `EmergencyProtectedTimelock`.
2. Cancelling all proposals that have not been executed.
3. Setting the `governance` address to a pre-configured Emergency Governance address. In the simplest scenario, this would be the Lido DAO Aragon Voting contract.

### Function: EmergencyProtectedTimelock.submit

```solidity
function submit(address executor, ExecutorCall[] calls)
  returns (uint256 proposalId)
```

Registers a new governance proposal composed of one or more EVM `calls` to be made by the `executor` contract.

#### Returns

The ID of the successfully registered proposal.


#### Preconditions

* MUST be called by the `governance` address.


### Function: EmergencyProtectedTimelock.schedule

```solidity
function schedule(uint256 proposalId)
```

#### Preconditions

* MUST be called by the `governance` address.
* The proposal MUST be already submitted.
* The post-submit timelock MUST already elapse since the moment the proposal was submitted.


### Function: EmergencyProtectedTimelock.execute

```solidity
function execute(uint256 proposalId)
```

Instructs the executor contract associated with the proposal to issue the proposal's calls.

#### Preconditions

* Emergency mode MUST NOT be active.
* The proposal MUST be already submitted & scheduled for execution.
* The emergency protection delay MUST already elapse since the moment the proposal was scheduled.

### Function: EmergencyProtectedTimelock.cancelAllNonExecutedProposals

```solidity
function cancelAllNonExecutedProposals()
```

Cancels all non-executed proposal, making them forever non-executable.

#### Preconditions

* MUST be called by the `governance` address.

### Function: EmergencyProtectedTimelock.activateEmergencyMode

```solidity
function activateEmergencyMode()
```

Activates the Emergency Mode.

#### Preconditions

* MUST be called by the Emergency Activation Committee address.
* The Emergency Mode MUST NOT be active.

### Function: EmergencyProtectedTimelock.emergencyExecute

```solidity
function emergencyExecute(uint256 proposalId)
```

Executes the scheduled proposal, bypassing the post-schedule delay.

#### Preconditions

* MUST be called by the Emergency Execution Committee address.
* The Emergency Mode MUST be active.

### Function: EmergencyProtectedTimelock.deactivateEmergencyMode

```solidity
function deactivateEmergencyMode()
```

Deactivates the Emergency Activation and Emergency Execution Committees (setting their addresses to `0x00`), cancels all unexecuted proposals, and disables the [Protected deployment mode](#Proposal-execution-and-deployment-modes).

#### Preconditions

* The Emergency Mode MUST be active.
* If the Emergency Mode was activated less than the `emergency mode max duration` ago, MUST be called by the [Admin Executor](#Administrative-actions) address.

### Function: EmergencyProtectedTimelock.emergencyReset

```solidity
function emergencyReset()
```

Resets the `governance` address to the `EMERGENCY_GOVERNANCE` value defined in the configuration, cancels all unexecuted proposals, and disables the [Protected deployment mode](#Proposal-execution-and-deployment-modes).

#### Preconditions

* The Emergency Mode MUST be active.
* MUST be called by the Emergency Execution Committee address.

### Admin functions

The contract has the interface for managing the configuration related to emergency protection (`setEmergencyProtection`) and general system wiring (`transferExecutorOwnership`, `setGovernance`). These functions MUST be called by the [Admin Executor](#Administrative-actions) address, basically routing any such changes through the Dual Governance mechanics.


## Contract: GateSealBreaker.sol

In the Lido protocol, specific critical components (`WithdrawalQueue` and `ValidatorsExitBus`) are safeguarded by the `GateSeal` contract instance. According to the gate seals [documentation](https://github.com/lidofinance/gate-seals?tab=readme-ov-file#what-is-a-gateseal):

>*"A GateSeal is a contract that allows the designated account to instantly put a set of contracts on pause (i.e. seal) for a limited duration.  This will give the Lido DAO the time to come up with a solution, hold a vote, implement changes, etc.".*

However, the effectiveness of this approach is contingent upon the predictability of the DAO's solution adoption timeframe. With the dual governance system, proposal execution may experience significant delays based on the current state of the `DualGovernance` contract. There's a risk that `GateSeal`'s pause period may expire before the Lido DAO can implement the necessary fixes.

To address this compatibility challenge between gate seals and dual governance, the `GateSealBreaker` contract is introduced. The `GateSealBreaker` enables the trustless unpause of contracts sealed by a `GateSeal` instance, but only under specific conditions:
- The minimum delay defined in the `GateSeal` contract has elapsed.
- Proposal execution is allowed within the dual governance system.

For seamless integration with the `DualGovernance` and `GateSealBreaker` contracts, the `GateSeal` instance will be configured as follows:

- `MAX_SEAL_DURATION_SECONDS` and `SEAL_DURATION_SECONDS` are set to `type(uint256).max`, what equivalent to `PAUSE_INFINITELY`, for the [PausableUntil.sol](https://github.com/lidofinance/core/blob/master/contracts/0.8.9/utils/PausableUntil.sol) contract.
- `MIN_SEAL_DURATION_SECONDS` is set to a finite duration, allowing the Lido DAO sufficient time to respond and adopt proposals when the `DualGovernance` contract is in the `Normal` state.

With such settings, the `GateSeal` instance seals the contracts indefinitely. However, anyone can initiate the process of "breaking the seal" by calling the `GateSealBreaker.startRelease(address gateSeal)` function, provided both requirements are met:

- The `MIN_SEAL_DURATION_SECONDS` has elapsed since the committee activated the `GateSeal`.
- The `DualGovernance` is currently in the `Normal` or `VetoCooldown` state, allowing proposals scheduling.

The `GateSealBreaker.startRelease()` function can be called only once for each activated `GateSeal` contract registered in the `GateSealBreaker`. This function effectively begins the countdown to release the seal, starting the `RELEASE_DELAY`.

During the `RELEASE_DELAY`, the sealed contracts remain paused, providing the Lido DAO time to schedule proposals within the dual governance system (the scheduling is allowed, which is guaranteed by the governance state precondition of the `GateSealBreaker.startRelease` function).

Upon completion of the `RELEASE_DELAY`, the `GateSealBreaker.enactRelease(address gateSeal)` function can be called to unpause the sealed contracts. This function is trustless and may only be called once. It does not revert even if some or all attempts to unpause the sealed contracts fail.

### Function GateSealBreaker.registerGateSeal

```solidity
function registerGateSeal(IGateSeal gateSeal)
```

This function should be invoked by the Lido DAO during the setup of the `GateSeal` instance. Upon registration in the contract, an activated `GateSeal` instance becomes eligible for release using the `startRelease()`/`enactRelease()` methods.

#### Preconditions

- MUST be called by the contract owner (supposed to be set to Lido DAO).
- The `GateSeal` instance being registered MUST NOT have been previously registered.

### Function GateSealBreaker.startRelease

```solidity
function startRelease(IGateSeal gateSeal)
```

Initiates the release process for the activated `GateSeal` instance registered in the contract. Records the release initiation timestamp and starts the `RELEASE_DELAY` period for the specific `gateSeal`.

#### Preconditions

- The specified `gateSeal` MUST be registered in the contract.
- The `gateSeal` MUST be activated by the gate seal committee.
- The `MIN_SEAL_DURATION_SECONDS` MUST have passed since the activation of the `gateSeal`.
- The `gateSeal` MUST NOT be already released.
- The `DualGovernance` contract MUST be in either the `Normal` or `VetoCooldown` state.

### Function GateSealBreaker.enactRelease

```solidity
function enactRelease(IGateSeal gateSeal)
```

Unpauses all contracts sealed by the specified `gateSeal` once the `RELEASE_DELAY` has elapsed since the release initiation.

Retrieves all sealed contracts via the `GateSeal.sealed_sealables()` view function and calls `IPausableUntil(sealable).resume()` for each sealed contract.

If any call to a sealable, including the `resume()` call, fails during the execution, the transaction WILL NOT revert but will emit the `ErrorWhileResuming(sealable, lowLevelError)` event for each contract that failed to unpause.

#### Preconditions

- The `GateSealBreaker.startRelease()` function MUST be called for the specified `gateSeal`.
- The `RELEASE_DELAY` for the specified `gateSeal` MUST have elapsed since the release initiation.
- The `GateSealBreaker` contract SHOULD have been granted rights to unpause the sealed contracts.

## Contract: Configuration.sol

`Configuration.sol` is the smart contract encompassing all the constants in the Dual Governance design & providing the interfaces for getting access to them. It implements interfaces `IAdminExecutorConfiguration`, `ITimelockConfiguration`, `IDualGovernanceConfiguration` covering for relevant "parameters domains".


## Upgrade flow description

In designing the dual governance system, ensuring seamless updates while maintaining the contracts' immutability was a primary consideration. To achieve this, the system was divided into three key components: `DualGovernance`, `EmergencyProtectedTimelock`, and `Executor`.

When updates are necessary only for the `DualGovernance` contract logic, the `EmergencyProtectedTimelock` and `Executor` components remain unchanged. This simplifies the process, as it only requires deploying a new version of the `DualGovernance`. This approach preserves proposal history and avoids the complexities of redeploying executors or transferring rights from previous instances.

During the deployment of a new dual governance version, the Lido DAO will likely launch it under the protection of the emergency committee, similar to the initial launch (see [Proposal execution and deployment modes](#Proposal-execution-and-deployment-modes) for the details). The `EmergencyProtectedTimelock` allows for the reassembly and reactivation of emergency protection at any time, even if the previous committee's duration has not yet concluded.

A typical proposal to update the dual governance system to a new version will likely contain the following steps:

1. Set the `governance` variable in the `EmergencyProtectedTimelock` instance to the new version of the `DualGovernance` contract.
2. Update the implementation of the `Configuration` proxy contract if necessary.
3. Configure emergency protection settings in the `EmergencyProtectedTimelock` contract, including the address of the committee, the duration of emergency protection, and the duration of the emergency mode.

For more significant updates involving changes to the `EmergencyProtectedTimelock` or `Proposals` mechanics, new versions of both the `DualGovernance` and `EmergencyProtectedTimelock` contracts are deployed. While this adds more steps to maintain the proposal history, such as tracking old and new versions of the Timelocks, it also eliminates the need to migrate permissions or rights from executors. The `transferExecutorOwnership()` function of the `EmergencyProtectedTimelock` facilitates the assignment of executors to the newly deployed contract.
