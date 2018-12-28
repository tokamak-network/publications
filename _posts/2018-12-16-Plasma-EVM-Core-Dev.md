---
layout: post
title:  Plasma EVM Core Dev Call User Activated Fork
date:   2018-12-16 00:00:00 +09:00
author: "Onther"
categories: ethereum scalability plasma plasmaEVM DevCall
tag: [ethereum, scalability, plasma, plasmaEVM, DevCall]
youtubeId : ikIhihHyCK0
slideWebId :
use_math : true
---
* content
{:toc}
{% include youtubePlayer.html id=page.youtubeId %}
{% include slidePlayer.html id=page.slideWebId %}

# Abstract

_User-activated fork_ (UAF) makes sure that users can protect assets when operator withholds blocks. When user prepare _User-submitted request epoch_ (URE) and submit _user-submitted block_ (URB) based on last finalized block, the canonical chain is forked to new chain from the last finalized block. In the new chain, all epochs after finalized epoch (epoch whose last block is finalized) should be rebased (like git but the commit sequence is not same) to the new fork.

When child chain is forked, new **3** epochs are placed as this sequence.
`last finalized epoch - URE - not finalized ORE' - not finalized NRE'`
Users can protect their asset unless withheld NRBs are not finalized. Exit requests for ORB are not rebased into the new fork to prevent them from being challenged because exit requests for URB MUST be created in case of operator's block withholding attack.

Users who want to exit should recreate exit request if child chain is forked.

# User-activated Fork

User can create and submit URB, which is more expensive than ORB, to fork child chain if he is willing to. User have to prepare URE before submit URB. In the prepare step, the RootChain contract emits `EpochPrepared(bool isRequest, bool userActivated, uint requestStart, uint requestEnd)` event. `requestStart` is the first id of ERU (exit request and undo request for URB) to be applied, and `first id = last id of previous URE if there was fork else 0`.

We can say an ERU is valid if 1) it is exit request and exit request transaction is not reverted or 2) it is undo request and the corresponding enter request is not applied yet in child chain.

When URE is prepared, anyone can submit URBs to fill the epoch with bond as Ether.

After URE, two epochs, ORE' _without exit requests_ and NRE', are placed. ORE' is a single epoch that covers all ORE in previous chain and so is NRE'. Because previous exit requests for ORB may not be valid after URE, they should not be included in ORE', but user can exit in URB by creating exit request for URB (ERU). User who want to exit have to make an exit request for URB if he notices BWA. If he think operator is honest, he will make another exit request for ORB.

# Undo Request

Undo request prevents future enter request in child chain from being applied. When undo request is finalized in root chain, state change is rewound and the undo request bond is refunded if enter request is not finalized yet. If the enter request is aleady finalized, there is no rewinding and no refund because the undo request is invalid. User shall create undo request only if he is convinced of user-activated fork.

-   $$r_e$$: Enter Request for ORB

-   $$r_u$$: Undo Request of $$r_e$$ for URB

-   $$r_u$$ is created after $$r_e$$ is in root chain

-   $$r_u$$ is applied **before** $$r_e$$ is in child chain

-   **$$r_u$$ cancels the future $$r_e$$.**

-   $$r_u$$ \_MUST_ have same $$trieKey$$ and $$trieValue$$ as $$r_e$$, and it is enforced by the RootChain contract

User creates $$r_u$$ when he want to cancel the previous $$r_e$$. $$r_u$$ is applied before $$r_e$$ if child chain is forked. $$r_u$$ is invalid if child chain is not forked and $$r_e$$ is applied.

When $$r_u$$ is finalized, state change, that is already applied in root chain, should be rewound. The finalization means that $$r_e$$ in child chain cannot be applied anymore, because $$r_u$$ is prior to $$r_e$$ by the UAF as opposed to the sequence in root chain. If $$r_e$$ is already applied in child chain, $$r_u$$ _MUST_ not rewind the state change and _MUST_ not refund request bond.

## Pseudocode: SampleRequestableContract

```solidity
contract SampleRequestableContract {
  mapping (uint => bool) public appliedRequests;

  // actually, this represents "ERU"
  mapping (uint => bool) public appliedUndoRequests;

  // requests to be undone
  mapping (uint => bool) public undoneRequests;

  function applyRequest(
    bool isExit, bool isRootChain
    uint256 requestId, address requestor,
    bytes32 trieKey, bytes32 trieValue
  ) external returns (bool success) {
    ... // check msg.sender is rootchain or null address

    // check r_e
    require(!appliedRequests[requestId]);

    // revert if r_u is already applied in child chain
    // Or we may just return false with emitting AlreadyUndoneRequest event.
    if (!isRootChain) {
      require(!undoneRequests[requestId]);
    }

    ... // do something

    appliedRequests[requestId] = true;
  }

  // undo a FUTURE request
  function undoRequest(
    // bool isExit,               // undo only enter request
    bool isRootChain,
    uint256 targetRequestId,              // r_e, request to be undone
    uint256 requestId, address requestor, // r_u, undo request
    bytes32 trieKey, bytes32 trieValue
  ) external returns (bool success) {
    ... // check msg.sender is rootchain or null address

    // check r_u
    require(!appliedUndoRequests[requestId]);

    // check r_e
    require(!undoneRequests[targetRequestId]);

    if (isRootChain) {
      // r_e should be already applied in root chain
      require(appliedRequests[targetRequestId]);

      ... // rewind the previous change
    } else {
      // r_e should not be already applied in root chain
      require(!appliedRequests[targetRequestId]);

      // do nothing
    }

    appliedUndoRequests[requestId] = true;
    undoneRequests[targetRequestId] = true;
  }
}
```

# Rebase and Resubmit URBs

UAF requires child chain to be rebased as follow.

`last finalized epoch - URE - not finalized ORE' - not finalized NRE'`

If user create and submit another URB before previous URB is finalized, next chain is rebased based on `not finalized ORE's`. Any user can submit `ORB's` to the RootChain. ORB' submiter may be incentivised from the URB submit cost. But details will be covered from Plasma EVM economic paper.

| Action              | Valid canonical chain                                                       |
| ------------------- | --------------------------------------------------------------------------- |
| Initial State       | Finalized Block#1 - ORB#2 - NRB#3 - ORB#4 - NRB#5 - ...                     |
| UAF 1               | Block#1 - **URB#2 - ORB'#3 - NRB'#4**                                       |
| extend forked chain | Block#1 - URB#2 - ORB'#3 - NRB'#4 - **ORB#5 - NRB#6 - ORB#7 - NRB#8** - ... |
| UAF 2               | Block#1 - URB#2 - ORB'#3 - **URB#4 - ORB'#5 - NRB'#6**                      |

Resubmitted URB and ORB' following the URB can be verified by verification game.

# Computation Challenge and Renew

Any computation challenge requires all blocks after the challenged block to be renewed their `stateRoot` and `receiptRoot`. A block is renewed with its 2 merkle roots and renewer.

If operator-submitetd block is challenged, we can even halt the chain and disincentivize her, but details will be covered from the economic paper. But in most case, she will withhold the block rather than broadcast invalid block.

If user-submitted block is challenged, other user can challenge on the block becuase user-submitted block data are available to anyone. We can disincentivize submiter and renew the block with the data based on challenger.

# Related Links
[Github Issue Link](https://github.com/Onther-Tech/research/blob/master/plasma-evm/2-user-activated-fork-and-undo-request.md)
