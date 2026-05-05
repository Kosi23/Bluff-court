# Developer Documentation

This document covers the Bluff Court Intelligent Contract — what it does, how it was built, how each function works, and how it was tested onchain.

## Overview

Bluff Court is a multiplayer social deduction game built as an Intelligent Contract on GenLayer. Players submit defenses to a shared prompt, but one player secretly receives a different prompt and is bluffing. AI validators reach consensus through Optimistic Democracy to identify the Liar, and the verdict is recorded onchain.

The contract showcases what Intelligent Contracts can do that traditional smart contracts cannot: make subjective judgments through AI consensus, with no oracle or centralized arbiter.

## Deployment

| Field | Value |
|---|---|
| Contract name | `BluffCourt` |
| File | `bluff_court.py` |
| Network | GenLayer Studionet |
| Deployed address | `0x66...a647` |
| Language | Python (Intelligent Contract) |
| Consensus method | Optimistic Democracy via `eq_principle.prompt_comparative` |

## State variables

| Variable | Type | Purpose |
|---|---|---|
| `shared_prompt` | `str` | The debate topic everyone is supposedly defending |
| `submissions` | `DynArray[str]` | List of all submitted defenses for the current round |
| `round_active` | `bool` | Whether a round is currently accepting submissions |
| `last_verdict` | `str` | Human-readable result of the last judged round |

## Functions

### Write methods (require transactions)

#### `start_round(prompt: str)`

Starts a new round. Clears any previous submissions and sets the debate topic.

- **Parameter:** `prompt` — the claim players will defend
- **Effect:** empties submissions, sets `shared_prompt`, sets `round_active` to `True`
- **Example:** `start_round("Pineapple belongs on pizza")`

#### `submit(defense: str)`

Adds a player's defense to the current round.

- **Parameter:** `defense` — the text the player is submitting
- **Validation:** throws `UserError` if no round is active or if 5 submissions are already received
- **Effect:** appends the defense to the submissions array

#### `judge_round()`

Sends all submissions to LLM validators and records their consensus on who the Liar is.

- **Validation:** throws `UserError` if no round is active or if fewer than 3 submissions exist
- **Effect:**
  1. Builds a prompt for the validators including the shared topic and all numbered submissions
  2. Calls `gl.eq_principle.prompt_comparative` to run multiple validators
  3. Validators independently identify which submission doesn't defend the claim
  4. Consensus result is parsed and stored in `last_verdict`
  5. Sets `round_active` to `False`

The validator prompt sent to the LLMs:

```
Players were each asked to defend the claim:
"{shared_prompt}"

One player secretly received a different prompt and is bluffing.
Read all submissions and identify which one does NOT actually engage 
with the claim and is therefore the Liar.

Submissions:
1. {submission 1}
2. {submission 2}
...

Respond with just a JSON object like: {"liar": <number>}
```

The Equivalence Principle used is `prompt_comparative` with the criterion *"The liar number must match"* — meaning all validators must agree on the same submission number for consensus to be reached.

### Read methods (no transaction, free)

#### `get_verdict() -> str`

Returns the human-readable result of the last judged round.

Possible return values:
- `"No round judged yet"` — initial state
- `"Round in progress, waiting for submissions"` — round is active, not yet judged
- `"Liar identified: submission N"` — round is complete

#### `get_submission_count() -> int`

Returns how many submissions have been received in the current round.

#### `get_prompt() -> str`

Returns the current debate topic.

## Build process

The contract was built in two main iterations.

### Iteration 1: Hardcoded proof of concept

The first version had five hardcoded submissions and a single `judge_round` function. The goal was to prove that LLM validators could identify a Liar at all before adding any complexity.

This version was tested with two scenarios:

**Test 1 — Obvious Liar:** Five submissions, where #5 was about cats instead of pizza.

**Test 2 — Sneaky Liar:** Five submissions, where #5 talked about pizza dough fermentation and the Maillard reaction but never actually defended pineapple. This tested whether validators were genuinely reasoning or just keyword matching.

Both tests resulted in correct identification of submission 5.

### Iteration 2: Dynamic submissions

The second version replaced hardcoded data with proper state management:
- `start_round` sets up a new round
- `submit` lets players add defenses one at a time
- `judge_round` works with whatever submissions are currently stored

This version was tested end-to-end with five separate `submit` transactions followed by `judge_round`. The validator consensus correctly identified submission 5 again.

## Test results

All test transactions reached `FINALIZED` status on GenLayer Studionet.

| Test | Setup | Result |
|---|---|---|
| 1 — Obvious Liar | Hardcoded, cats-instead-of-pizza | ✅ Submission 5 identified |
| 2 — Sneaky Liar | Hardcoded, pizza-dough-not-pineapple | ✅ Submission 5 identified |
| 3 — Dynamic flow | 5 separate submit calls + judge | ✅ Submission 5 identified |

## Why this works for the GenLayer mission

The mission brief asks for games that *"showcase GenLayer's Intelligent Contract and Optimistic Democracy consensus."* Bluff Court doesn't include consensus as a feature — consensus IS the gameplay. Without Optimistic Democracy, there's no way to fairly resolve "is this defense bluffing or genuine?" The game cannot exist on a traditional smart contract.

The brief also requires:
- **Multiplayer / in rooms** — supports up to 5 players per round
- **5–15 minute duration** — round resolution takes ~1–2 minutes; full session fits comfortably
- **Replayable weekly** — prompt rotation provides infinite content variation
- **Leaderboard for XP** — planned addition

## Known limitations

- **Liar selection is currently manual.** Whoever calls `start_round` decides the prompt. Future versions will randomly assign a Liar index and send that player a different prompt privately.
- **No player identity tracking yet.** Submissions are anonymous. There's no way to attribute defenses to specific players or track per-player win/loss records.
- **No XP or leaderboard logic in the contract yet.** Will be added once player tracking exists.
- **No frontend yet.** Players currently interact through GenLayer Studio's debug UI. A Next.js + custodial backend frontend is in development so casual players can join without wallets.

## Full contract code

```python
# { "Depends": "py-genlayer:1jb45aa8ynh2a9c9xn3b7qqh8sm5q93hwfp7jqmwsfhh8jpz09h6" }
from genlayer import *
import json


class BluffCourt(gl.Contract):
    shared_prompt: str
    submissions: DynArray[str]
    round_active: bool
    last_verdict: str
    
    def __init__(self):
        self.shared_prompt = ""
        self.round_active = False
        self.last_verdict = "No round judged yet"
    
    @gl.public.write
    def start_round(self, prompt: str) -> None:
        while len(self.submissions) > 0:
            self.submissions.pop()
        self.shared_prompt = prompt
        self.round_active = True
        self.last_verdict = "Round in progress, waiting for submissions"
    
    @gl.public.write
    def submit(self, defense: str) -> None:
        if not self.round_active:
            raise gl.vm.UserError("No active round. Call start_round first.")
        if len(self.submissions) >= 5:
            raise gl.vm.UserError("Round is full. 5 submissions already received.")
        self.submissions.append(defense)
    
    @gl.public.write
    def judge_round(self) -> None:
        if not self.round_active:
            raise gl.vm.UserError("No active round to judge.")
        if len(self.submissions) < 3:
            raise gl.vm.UserError("Need at least 3 submissions to judge.")
        
        numbered_subs = ""
        for i in range(len(self.submissions)):
            numbered_subs += f"{i+1}. {self.submissions[i]}\n"
        
        prompt = f"""
Players were each asked to defend the claim:
"{self.shared_prompt}"

One player secretly received a different prompt and is bluffing.
Read all submissions and identify which one does NOT actually engage 
with the claim and is therefore the Liar.

Submissions:
{numbered_subs}

Respond with just a JSON object like: {{"liar": <number>}}
"""
        
        def get_verdict():
            result = gl.nondet.exec_prompt(prompt)
            result = result.replace("```json", "").replace("```", "")
            return result
        
        result = gl.eq_principle.prompt_comparative(
            get_verdict, 
            "The liar number must match"
        )
        
        parsed = json.loads(result)
        self.last_verdict = f"Liar identified: submission {parsed['liar']}"
        self.round_active = False
    
    @gl.public.view
    def get_verdict(self) -> str:
        return self.last_verdict
    
    @gl.public.view
    def get_submission_count(self) -> int:
        return len(self.submissions)
    
    @gl.public.view
    def get_prompt(self) -> str:
        return self.shared_prompt
```
