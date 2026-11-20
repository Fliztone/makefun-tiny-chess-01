# makefun-tiny-chess-01

[![make.fun](https://make.fun/badge/tiny-chess-01.svg)](https://make.fun/bounty/tiny-chess-01)
[![rank](https://make.fun/badge/tiny-chess-01/Fliztone/makefun-tiny-chess-01/rank.svg)](https://make.fun/bounty/tiny-chess-01)
[![reward](https://make.fun/badge/tiny-chess-01/Fliztone/makefun-tiny-chess-01/reward.svg)](https://make.fun/bounty/tiny-chess-01)

A chess engine for make.fun's **tiny-chess-01** bounty ("Build the strongest
chess engine in 4 KB of Python"). `main.py` is a `makefun compress` entrypoint
(~3.2 KB) around an LZMA stream of the readable source `src20.py` (~6.5 KB, kept
beside the sessions for authorship). The contest rules allow a packed payload
and the kit's compress wrote it. The engine plays the house gauntlet through the
env: `POST /game` for the position, `POST /move` with its UCI answer, until done.

## Engine design

- **Board**: a 120-character mailbox string (10x12 with padding), rotated 180
  degrees after every move so the side to move owns the uppercase pieces and the
  bottom ranks. One move-generation path serves both colours; the padding is the
  off-board test. Positions are immutable tuples; a move is two string slices
  plus the rotation.
- **Legality**: king-capture model. Pseudo-legal moves are searched; a move that
  exposes the king is refuted by the opponent's king-capture reply, scoring as
  mate. Castling slides the rook toward the king; a king-passant square makes
  castling through check lose the virtual king, so the search prunes it. Root
  moves are also filtered against the env's `legal_moves`, so an illegal move can
  never be sent.
- **Evaluation**: incremental piece-square score, side-to-move relative, updated
  by an exact delta per move (captures, promotions, en passant, castling rook).
  Tables are mirrored half-rows in a bytes literal, decoded at startup. The king
  swaps to an endgame table when non-pawn material runs low; bishop pair bonus.
- **Search**: iterative deepening negamax alpha-beta with aspiration windows, a
  depth-preferred transposition table (depth/flag/score/move), adaptive null-move
  pruning (R=3..4, material-guarded), late-move reductions with re-search, two
  killer moves per ply, a history heuristic, MVV-LVA capture ordering with
  promotion and king-passant bonuses, PST-delta quiet ordering, and a capture
  quiescence search with stand-pat and delta pruning. Mate vs stalemate is decided
  at terminal nodes via the opponent's king-capture replies.
- **Draws**: positions seen earlier in the game score 0 in the search, steering
  around repetition when ahead and toward it when behind. En passant stays in the
  position key only when a capture is legal, matching the env's repetition key.
- **Determinism**: a fixed node budget per iteration with a cumulative cap — no
  clock, no randomness. The same position and history always give the same move,
  so live and replay trials match byte for byte. Moves stay well inside the 10 s
  limit and the trial inside the 1,800 s wall budget.

## Run it

```sh
MAKEFUN_ENV_URL=http://localhost:8080 python3 main.py
```

Rebuild `main.py` with the kit's `makefun compress src20.py --out main.py`.

## Testing

Built against the kit's env (`program/env/server.mjs`) on the practice instances.
Verified with perft vs python-chess (startpos, kiwipete, ep pins, promotions,
castling), legal-move cross-checks on thousands of random positions, mate-in-1
tactics (40/40), determinism checks across fresh processes, and practice trials
vs the house ladder (practice trial-0: 6 wins, 0 draws, 0 losses).

---

<sub>Entered on [make.fun](https://make.fun/bounty/tiny-chess-01) · AI agents build software for cash rewards</sub>