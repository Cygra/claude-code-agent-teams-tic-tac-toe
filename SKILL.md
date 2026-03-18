---
name: tic-tac-toe
description: This skill should be used when the user says "来一盘井字棋", "我要玩井字棋", "玩井字棋", "三连棋", "三目並べ", "○×ゲーム", "play tic-tac-toe", "let's play tic-tac-toe", "noughts and crosses", or any request in any language to play a game of Tic-Tac-Toe / 井字棋 / 三目並べ.
version: 0.1.0
allowed-tools: Bash, TeamCreate, Agent, SendMessage, TeamDelete
---

# Tic-Tac-Toe Skill

Two AI agents — X and O — play Tic-Tac-Toe on a 3×3 board. You (team-lead) orchestrate the game, display the board after every move, and referee the result.

---

## Step 1 — Prerequisites Check

Run this Bash command:

```bash
printenv CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS
```

If the output is NOT exactly `1`, output the following message and **stop immediately**:

```
⚠️  Tic-Tac-Toe requires the experimental Agent Teams feature.

To enable it, add the following to the "env" section of ~/.claude/settings.json:

  "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"

Then restart Claude Code and try again.
```

---

## Step 2 — Create the Team

Call `TeamCreate` with:
- `team_name`: `"tic-tac-toe"`
- `description`: `"Tic-Tac-Toe — X vs O"`

---

## Step 3 — Spawn Both Player Agents (in parallel)

In a **single message**, make two Agent tool calls simultaneously:

**Call 1 — x-player:**
- `subagent_type`: `"general-purpose"`
- `team_name`: `"tic-tac-toe"`
- `name`: `"x-player"`
- `prompt`: (see **Appendix A** below)

**Call 2 — o-player:**
- `subagent_type`: `"general-purpose"`
- `team_name`: `"tic-tac-toe"`
- `name`: `"o-player"`
- `prompt`: (see **Appendix B** below)

---

## Step 4 — Initialize the Board

The board is a **9-character flat string** (3 rows × 3 columns, row-major, top to bottom):

- `.` = empty cell
- `X` = X piece
- `O` = O piece
- Cell index formula: `index = row * 3 + col` (row 0 = top, row 2 = bottom; col 0..2 = left..right)
- Cell numbers shown to users: **1–9** (index = cell_number − 1)

```
Cell numbers:    Board indices:
 1 | 2 | 3       0 | 1 | 2
-----------     -----------
 4 | 5 | 6       3 | 4 | 5
-----------     -----------
 7 | 8 | 9       6 | 7 | 8
```

Initial board: `"........."` (exactly 9 dots)

Render and display the initial empty board to the user (see **Board Rendering** below), then proceed to the game loop.

---

## Step 5 — Game Loop

Track these variables:
- `board` — the current 9-char board string
- `move_count` — starts at 0, increments each move
- `current_player` — starts as `"x-player"` (X goes first)
- `current_piece` — starts as `"X"`
- `last_move_description` — starts as `"Game start"`

Repeat the following until `game_over`:

### 5a — Send board to current player

Call `SendMessage` with:
```
type: "message"
recipient: <current_player>
content:
  BOARD: <9-char board string>
  TURN: <move_count>
  YOUR_PIECE: <current_piece>
  OPPONENT_PIECE: <O if current_piece is X, else X>
  LAST_MOVE: <last_move_description>
summary: "Your turn — choose cell 1-9"
```

### 5b — Receive and parse the response

The player's reply will arrive automatically. Extract from the content:
- `MOVE: N` — the cell chosen (1–9)
- `BOARD: <9-char string>` — the updated board after placing their piece

### 5c — Validate the move

Check:
1. The new board string is exactly 9 characters.
2. The new board has exactly one more occurrence of `current_piece` compared to the old board.
3. All other characters are unchanged (no cell was overwritten).

If validation fails, re-send the original board with an error note:
```
ERROR: Your board update was invalid. The cell may already be occupied, or your string length is wrong.
BOARD: <original 9-char board>
TURN: <move_count>
YOUR_PIECE: <current_piece>
OPPONENT_PIECE: <opponent_piece>
LAST_MOVE: <last_move_description>
```

### 5d — Update state and render

Set `board` to the new board string.
Set `last_move_description` to `"X played cell N"` or `"O played cell N"`.
Increment `move_count`.
Render the board for the user (see **Board Rendering** below).

### 5e — Check for win

After each move, check whether `current_piece` occupies any winning line.

**All 8 winning combinations** (by board index):

| Line | Indices |
|------|---------|
| Top row | 0, 1, 2 |
| Middle row | 3, 4, 5 |
| Bottom row | 6, 7, 8 |
| Left column | 0, 3, 6 |
| Center column | 1, 4, 7 |
| Right column | 2, 5, 8 |
| Diagonal ↘ | 0, 4, 8 |
| Diagonal ↙ | 2, 4, 6 |

For each combination: if all three indices in `board` equal `current_piece` → **win detected**, proceed to **Step 6**.

### 5f — Check for draw

If `move_count == 9` and no win → **draw**, proceed to **Step 6**.

### 5g — Rotate player

Swap:
- `current_player`: `"x-player"` ↔ `"o-player"`
- `current_piece`: `"X"` ↔ `"O"`

Continue the loop.

---

## Board Rendering

To display the board, convert the 9-char string to a visual grid:

**Character mapping:**
- `.` → the cell number (1–9) in dim text, or `·` if you prefer
- `X` → `❌`
- `O` → `⭕`

**Format** (output this to the user after each move):

```
+-------+
| ❌ · ⭕ |
| · ❌ · |
| ⭕ · ❌ |
+-------+
Move 5 — ❌ X played cell 5
Next: ⭕ O's turn
```

For empty cells, display the cell number so players (and user) can see available positions:

```
+-------+
| 1  2  3 |
| 4  5  6 |
| 7  8  9 |
+-------+
🎮 Game started! ❌ X goes first.
```

After the first move, replace occupied cells with ❌/⭕ and show remaining empty cells as their numbers:

```
+-------+
| 1  2  3 |
| 4  ❌  6 |
| 7  8  9 |
+-------+
Move 1 — ❌ X played cell 5
Next: ⭕ O's turn
```

---

## Step 6 — End the Game

Display the final board, then announce the result:

- X win: `🎉 Game over! ❌ X wins in N moves!`
- O win: `🎉 Game over! ⭕ O wins in N moves!`
- Draw: `🤝 Game over! It's a draw!`

### Cleanup

Send shutdown requests to both players **in parallel** (single message, two SendMessage calls):
```
SendMessage type="shutdown_request" recipient="x-player" content="Game over, thanks for playing!"
SendMessage type="shutdown_request" recipient="o-player" content="Game over, thanks for playing!"
```

After both shutdown_response messages are received, call `TeamDelete`.

---

## Appendix A — x-player Prompt

Use this verbatim as the `prompt` parameter when spawning `x-player`:

```
You are x-player in a Tic-Tac-Toe game, part of team "tic-tac-toe". Your piece is X.

## Your Role
You wait for messages from team-lead. Each message contains the current board state and asks you to make a move. You choose an empty cell, place your piece, and reply with the updated board.

## Board Format
The board is a 9-character string (3 rows × 3 columns, row-major, top to bottom).
- Index formula: position = row * 3 + col  (row 0 = top, row 2 = bottom; col 0..2 = left..right)
- Characters: '.' = empty, 'X' = your piece, 'O' = opponent's piece
- Cell numbers 1-9 map to indices 0-8 (index = cell_number - 1)

Cell layout:
  Cell 1 (index 0) | Cell 2 (index 1) | Cell 3 (index 2)
  Cell 4 (index 3) | Cell 5 (index 4) | Cell 6 (index 5)
  Cell 7 (index 6) | Cell 8 (index 7) | Cell 9 (index 8)

## When You Receive a Message
team-lead will send you a message containing lines like:
  BOARD: <9-char string>
  YOUR_PIECE: X
  OPPONENT_PIECE: O

Do the following:
1. Parse the 9-character board string.
2. Decide which empty cell (1–9) to play. Use strategic reasoning in this priority order:
   a. WIN: if you can complete 3-in-a-row right now, do it.
   b. BLOCK: if the opponent can complete 3-in-a-row on their next turn, block that cell.
   c. CENTER: if cell 5 (index 4) is empty, take it.
   d. CORNERS: prefer corners (cells 1, 3, 7, 9) over edges (cells 2, 4, 6, 8).
   e. Otherwise pick any empty cell.
3. Verify your chosen cell is empty ('.') in the board string (index = cell - 1). If not, pick another empty cell.
4. Replace that '.' with 'X' in the 9-char string. All other characters must remain identical.
5. Double-check: your new board string is exactly 9 characters and has exactly one more 'X' than the original.

## Winning combinations (by index) to check for step 2a and 2b:
  Rows:    [0,1,2], [3,4,5], [6,7,8]
  Columns: [0,3,6], [1,4,7], [2,5,8]
  Diagonals: [0,4,8], [2,4,6]

## How to Respond
Send a message back to team-lead using SendMessage:

  type: "message"
  recipient: "team-lead"
  content:
    MOVE: <cell number 1-9>
    BOARD: <the new 9-character board string>
  summary: "Played cell N"

## Important Rules
- Never play in an occupied cell (non-'.').
- Your response MUST contain exactly one SendMessage call.
- Do not check for wins yourself — team-lead handles that.
- Do not ask questions or add commentary — just play your move and reply.
- After replying, go idle and wait for the next message from team-lead.
- If you receive a shutdown_request, respond with a shutdown_response approving shutdown.
```

---

## Appendix B — o-player Prompt

Use this verbatim as the `prompt` parameter when spawning `o-player`:

```
You are o-player in a Tic-Tac-Toe game, part of team "tic-tac-toe". Your piece is O.

## Your Role
You wait for messages from team-lead. Each message contains the current board state and asks you to make a move. You choose an empty cell, place your piece, and reply with the updated board.

## Board Format
The board is a 9-character string (3 rows × 3 columns, row-major, top to bottom).
- Index formula: position = row * 3 + col  (row 0 = top, row 2 = bottom; col 0..2 = left..right)
- Characters: '.' = empty, 'X' = opponent's piece, 'O' = your piece
- Cell numbers 1-9 map to indices 0-8 (index = cell_number - 1)

Cell layout:
  Cell 1 (index 0) | Cell 2 (index 1) | Cell 3 (index 2)
  Cell 4 (index 3) | Cell 5 (index 4) | Cell 6 (index 5)
  Cell 7 (index 6) | Cell 8 (index 7) | Cell 9 (index 8)

## When You Receive a Message
team-lead will send you a message containing lines like:
  BOARD: <9-char string>
  YOUR_PIECE: O
  OPPONENT_PIECE: X

Do the following:
1. Parse the 9-character board string.
2. Decide which empty cell (1–9) to play. Use strategic reasoning in this priority order:
   a. WIN: if you can complete 3-in-a-row right now, do it.
   b. BLOCK: if the opponent can complete 3-in-a-row on their next turn, block that cell.
   c. CENTER: if cell 5 (index 4) is empty, take it.
   d. CORNERS: prefer corners (cells 1, 3, 7, 9) over edges (cells 2, 4, 6, 8).
   e. Otherwise pick any empty cell.
3. Verify your chosen cell is empty ('.') in the board string (index = cell - 1). If not, pick another empty cell.
4. Replace that '.' with 'O' in the 9-char string. All other characters must remain identical.
5. Double-check: your new board string is exactly 9 characters and has exactly one more 'O' than the original.

## Winning combinations (by index) to check for step 2a and 2b:
  Rows:    [0,1,2], [3,4,5], [6,7,8]
  Columns: [0,3,6], [1,4,7], [2,5,8]
  Diagonals: [0,4,8], [2,4,6]

## How to Respond
Send a message back to team-lead using SendMessage:

  type: "message"
  recipient: "team-lead"
  content:
    MOVE: <cell number 1-9>
    BOARD: <the new 9-character board string>
  summary: "Played cell N"

## Important Rules
- Never play in an occupied cell (non-'.').
- Your response MUST contain exactly one SendMessage call.
- Do not check for wins yourself — team-lead handles that.
- Do not ask questions or add commentary — just play your move and reply.
- After replying, go idle and wait for the next message from team-lead.
- If you receive a shutdown_request, respond with a shutdown_response approving shutdown.
```
