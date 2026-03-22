# Chess Analyzer Implementation Plan

**Project:** Chess.com Game Fetcher & Visual Analyzer
**Target User:** Keith
**Core Tech Stack:** Python, `requests` (Chess.com API), `python-chess` (PGN parsing & board visualization)

## Objective
Build an automated pipeline that pulls your recent chess games from the public Chess.com API, parses the PGN, generates visual board states for every single move using `python-chess`, and provides move-by-move commentary.

## Phase 1: Data Acquisition (Chess.com API)
- **Endpoint:** Use the PubAPI to fetch archives: `https://api.chess.com/pub/player/{username}/games/archives`
- **Target:** Pull the most recent month's archive, filter for the last completed game (or allow passing a specific game URL/ID).
- **Extraction:** Grab the raw `pgn` string from the game JSON payload.

## Phase 2: Game Parsing & Visualization (`python-chess`)
- **Dependencies:** `chess`, `chess.pgn`, `chess.svg`, and an SVG-to-PNG converter (like `cairosvg` or `svglib`) so the images can be easily embedded in standard markdown/chat interfaces.
- **Workflow:**
  1. Load the PGN string into a `chess.pgn.Game` object.
  2. Initialize an empty board state.
  3. Iterate through `game.mainline_moves()`.
  4. After each push, use `chess.svg.board(board)` to generate an SVG of the current position.
  5. Save the image sequentially (e.g., `move_001_w.png`, `move_001_b.png`).
  6. Highlight the most recent move on the board using the `arrows` or `squares` parameters in `chess.svg`.

## Phase 3: Commentary Engine
*Commentary requires evaluating the moves. We have two options here to combine or choose between:*
- **Option A (Algorithmic / Stockfish):** Hook `python-chess` up to a local Stockfish binary. Calculate the centipawn evaluation before and after the move. If the eval drops drastically, flag it as a "Blunder" and generate a snarky comment. If it's the top engine line, flag as "Best Move."
- **Option B (LLM Narrative):** Feed the PGN (or subsets of the move list) to an LLM (like Gemini or Claude) along with the Stockfish evaluations. Ask the LLM to write human-like, narrative commentary for the pivotal moments of the game.

## Phase 4: Assembly & Output
- **Markdown Report Generator:** Stitch the generated images and the commentary text into a chronological Markdown file.
- **Structure:**
  - Game Metadata (White vs Black, Date, Opening Name, Result)
  - Move-by-Move breakdown:
    - **Move 12. Nxf7**
    - *Image of the board state*
    - *Commentary:* "A devastating fork, or at least it would have been if your queen wasn't hanging."

## Phase 5: Automation / Next Steps
- Wrap the entire script into a CLI command: `python analyze.py --user keith --latest`
- Add support for uploading the report directly to a GitHub repo or sending it via a messaging integration.

