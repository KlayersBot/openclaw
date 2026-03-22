# Chess Analyzer Implementation Plan (Revised)

**Project:** Chess.com Game Fetcher & Visual Analyzer
**Target User:** Keith
**Core Tech Stack:** Python, `requests` (Chess.com API), `python-chess` (PGN parsing & board visualization), `pytest` (Testing)

## Objective
Build a resilient, automated pipeline that pulls your recent chess games from the public Chess.com API, parses the PGN, generates visual board states for every single move using `python-chess`, and provides move-by-move commentary. 

---

## Phase 1: Data Acquisition (Chess.com API)
- **Endpoint:** Use the PubAPI to fetch archives: `https://api.chess.com/pub/player/{username}/games/archives`
- **Target:** Pull the most recent month's archive, filter for the last completed game.
- **Hygiene & Error Handling:**
  - Implement exponential backoff for HTTP 429 (Rate Limit) and 5xx errors.
  - Set a custom `User-Agent` string (Chess.com API strictly requires this or they will block the request).
  - Handle "Aborted" games (games with 0 or 1 move) by skipping to the next valid completed game.

## Phase 2: Game Parsing & Visualization (`python-chess`)
- **Dependencies:** `chess`, `chess.pgn`, `chess.svg`, and `cairosvg` (for converting SVG strings to PNG bytes).
- **Workflow:**
  1. Load the PGN string into a `chess.pgn.Game` object.
  2. Iterate through `game.mainline_moves()`.
  3. After each push, use `chess.svg.board(board)` to generate an SVG.
  4. Convert the SVG to PNG and save it (e.g., `assets/move_001_w.png`).
- **Edge Cases to Handle:**
  - Verify castling, *en passant*, and underpromotions render correctly in `chess.svg`.
  - Flip the board perspective based on which color Keith played (Black vs. White).
  - Highlight the most recent move using the `arrows` or `squares` parameters.

## Phase 3: Commentary Engine
*Hybrid Approach:*
- **Algorithmic Baseline:** Hook `python-chess` to a local Stockfish binary. Calculate the centipawn evaluation difference between moves. Classify moves (Blunder, Mistake, Inaccuracy, Excellent, Best).
- **LLM Narrative:** Feed the PGN subset and Stockfish evaluations to an LLM (e.g., Gemini/Claude). Ask the LLM to write snarky, human-like commentary specifically roasting blunders or praising brilliancies. 
- **Hygiene:** Set strict timeout limits on the Stockfish evaluation (e.g., 0.1 seconds per move) to prevent the pipeline from hanging on long games.

## Phase 4: Assembly & Output
- **Markdown Report Generator:** Stitch the generated images and the commentary text into a chronological Markdown file.
- **Structure:**
  - Game Metadata (White vs Black, Date, Opening Name, Result, Accuracy)
  - Move-by-Move Breakdown
- **Hygiene:** Ensure image paths are relative so the Markdown renders correctly when pushed to GitHub.

## Phase 5: Automation & Deployment
- Wrap the entire script into a CLI command: `python analyze.py --user keith --latest`
- Add a GitHub Actions workflow or a cron job to automatically run this script daily, generate the report, and commit it to the repository.

---

## Phase 6: Testing & Quality Assurance (Hygiene)

To ensure the analyzer doesn't break silently or generate garbage outputs, a robust testing suite using `pytest` is required.

### 1. Unit Tests
- **API Fetcher:** Mock the `requests` library to simulate Chess.com API responses (including 404s, 429s, and malformed JSON) without hitting the actual rate limits.
- **PGN Parser:** Feed known, hardcoded PGN strings (including edge cases like underpromotions or games ending in draw by repetition) and assert the correct number of moves are parsed.
- **Stockfish Integration:** Pass a known blunder position to the engine wrapper and assert that the centipawn drop exceeds the blunder threshold.

### 2. Integration Tests
- Run the full pipeline (API -> PGN -> SVG -> Report) using a mocked API response.
- **Output Validation:** Check that the final Markdown file exists, contains the correct metadata, and references the exact number of expected PNG files.

### 3. Visual Regression / Snapshot Testing
- Save a "golden" set of SVG/PNG outputs for a specific known game.
- During testing, generate the images for that game and do a byte-comparison (or perceptual hash) against the golden set to ensure `chess.svg` updates or library changes haven't broken board rendering.

### 4. Continuous Integration
- Configure a GitHub Action (`.github/workflows/test.yml`) to run the `pytest` suite on every push or Pull Request to the repository.
