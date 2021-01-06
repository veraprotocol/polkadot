# Dispute Participation

Tracks all open disputes in the active leaves set.

## Remarks

`UnsignedTransaction`s are OK to be used, since the inner
element `CommittedCandidateReceipt` is verifiable.

## Assumptions

* If each validator gossips its own vote regarding the disputed block, upon receiving the first vote (the initiating dispute vote) there is only
a need for a very small gossip only containing the `Vote` itself.

## IO

Inputs:

* `DisputeEvent::DisputeVote`
* `DisputeEvent::DisputeLateTimeout`
* `DisputeEvent::DisputeResolution`

Outputs:

* `VotesDbMessage::StoreVote`
* `AvailabilityRecoveryMessage::RecoverAvailableData`
* `CandidateValidationMessage::ValidateFromChainState`

## Helper Structs

```rust
pub enum Vote {
	/// Fragment information of a `BackedCandidate`.
	Backing {
		/// The required checkable signature for the statement.
		attestation: ValidityAttestation,
		/// Validator index of the backing validator in the validator set
		/// back then in that session.
		validator_index: ValidatorIndex,
		/// Committed candidate receipt for the candidate.
		candidate_receipt: CommittedCandidateReceipt,
	},
	/// Result of secondary checking the dispute block.
	ApprovalCheck {
		/// Signed full statement to back the vote.
		sfs: SignedFullStatement
	},
	/// An explicit vote on the disputed candidate.
	DisputePositive {
		/// Signed full statement to back the vote.
		sfs: SignedFullStatement
	},
	/// An explicit vote on the disputed candidate.
	DisputeNegative {
		/// Signed full statement to back the vote.
		sfs: SignedFullStatement
	},
}
```

```rust
/// Resolution of a vote:
enum Resolution {
    /// The block is valid
    Valid,
    /// The block is valid
    Invalid,
}
```

## Constants

* `DISPUTE_POST_RESOLUTION_TIME = Duration::from_secs(24 * 60 * 60)`: Keep a dispute open for an additional time where validators
can cast their vote.

## Session Change

Operation resumes, since the validator set at the time of the dispute
stays operational, and the validators are still bonded.

## Storage

Nothing to store, everything lives in `VotesDB`.

## Sequences

### ViewChange is BlockImport

1. Extract imported blocks from `ViewChange`
1. for each imported block:
  1. Extract the associated dispute _runtime_ events
  1. for each runtime event:
  	1. Store vote according to the runtime events to the votesdb via `VotesDBMessage::StoreVote`
  	1. iff vote is new
	  1. add event to transaction events

The `VotesDB` is then responsible for emitting `Detection`, `Resolution` or `Timeout` messages.

#### Detection

The first negative vote will trigger the dispute detection.

1. Receive `DisputeParticipationMessage::DisputeDetection`
1. Request `AvailableData` via `RecoverAvailableData` to obtain the `PoV`.
1. Call `ValidateFromChainState` to validate the `PoV` with the `CandidateDescriptor`.
  1. Cast vote according to the `ValidationResult`.
  1. Store our vote to the `VotesDB` via `VotesDBMessage::StoreVote`

#### Resolution

1. Receive `DisputeParticipationMessage::DisputeResolution`
1. Craft an unsigned transaction with all votes cast
  1. includes `CandidateReceipts`
  1. includes `Vote`s
1. Store the resolution time.
1. Start the post resolution timeout.

### Blacklisting bad block and invalid decendants