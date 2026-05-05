# Bluff Court

A multiplayer social deduction game built on GenLayer. Players defend a shared claim, but one is secretly bluffing — and AI validators reach onchain consensus on who's lying.

Built for the GenLayer Foundation's Mini-games for the Community mission.

## What makes it interesting

Bluff Court isn't a game with blockchain stapled on. The consensus mechanism *is* the gameplay. There's no objectively correct answer to "is this defense genuine or a bluff?" — only consensus can fairly resolve it.

The game cannot exist on a traditional smart contract. That's the point.

## Quick start

1. Open [GenLayer Studio](https://studio.genlayer.com)
2. Create a new file called `bluff_court.py`
3. Copy the contract code from `DOCS.md`
4. Click Deploy
5. Call `start_round` → `submit` (5 times) → `judge_round` → `get_verdict`

## Documentation

See [DOCS.md](./DOCS.md) for full contract documentation, function reference, test results, and source code.

## Test results

Three rounds tested onchain with successful consensus:

- **Obvious Liar** (cats-instead-of-pizza) — caught ✅
- **Sneaky Liar** (pizza-dough but not pineapple) — caught ✅
- **Dynamic round** (5 separate `submit` calls) — caught ✅

## What's still in progress

- Frontend (Next.js + custodial backend so players don't need wallets)
- Random Liar selection in the contract
- Per-player tracking and leaderboard
- Random prompt rotation for weekly replayability

## Built for

The GenLayer Mini-games for Community mission, with an accompanying tutorial submitted to the Educational Content mission.

## License

MIT
