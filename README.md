# Pre-Teekathon Submission

Track 1: LLM Solver Program Synthesis.

## Live Demo

- Frontend: https://teekathon-frontend.vercel.app
- Backend: https://web-production-a19aa.up.railway.app
- Repo(s): https://github.com/Philbert51/teekathon-frontend, https://github.com/Philbert51/teekathon-backend
teekathon-frontend and teekathon-backend are deployment repos, containing only the files needed for Vercel and Railway respectively. Development happened in a local working directory; these are synced subsets of it

## Local Setup

Backend:

pip install -r requirements.txt

Set `GOOGLE_GENAI_API_KEY` in a `.env` file, then:

python simulator.py

Runs on `http://0.0.0.0:8000` by default; override with the `PORT` environment variable.

Frontend:
Open `index.html` in a browser, or serve the directory with any static file server. 
Update `backendUrl` in `index.js` to point at your running backend.

RGD requires the C++ planner built locally first (Windows: MSVC via CMake, see build notes in the original cpp/README.md; deployed backend builds it automatically via Dockerfile on Linux). Without a built run_planner, BFS, DFS, and LLM modes still work normally.
The run planner executable must be inside the relative folder where simulator.py exist

## Architecture

The frontend is DeepMind's original PushWorld play page, extended with solver controls, a live output log, and a solution playback panel. 
It fetches puzzle data live from the `google-deepmind/pushworld` GitHub repository.

The backend is a single Flask application. A POST to `/` accepts a puzzle and an algorithm choice, stores it under a generated run id, and returns that id. The frontend then opens a Server-Sent Events connection to `/receive/<run_id>`, which starts the chosen solver on a background thread and streams its progress one line per message until the run completes. This is what satisfies the real-time visualisation requirement: progress, not just a final answer, reaches the browser as it happens.

## Classical Solver Integration

BFS and DFS are implemented directly in Python, sharing simulation code `simulateStep`, `getPushedObjects` with the puzzle-loading step. Both use a precomputed offset-based collision table built once per puzzle at load time, rather than per-move pixel comparisons, since profiling an early version showed collision detection as the dominant cost.

RGD is DeepMind's original C++ planner from the `cpp` directory, invoked as a subprocess against a temporary `.pwp` file.

## LLM Harness

Given a puzzle format description, a solver interface (`def solve(puzzle): ... return "UDLR..."`), and one worked example, the harness asks an LLM Gemini, with several model variants tried in sequence as fallback to write a Python program. That program is written to a temp file with a small driver appended, executed as a subprocess against a real puzzle, and its output is replayed through the same `simulateStep`/collision logic used by BFS and DFS so a program is validated against the actual simulator, not against its own claims.

On failure : wrong action character, illegal push, wrong final state, non-zero exit, or timeout. The specific reason is sent back to the model along with `previous_interaction_id`, so the next attempt has the prior program and knows exactly what to fix. Up to 3 attempts per stage.

Curriculum mode solves a fixed sequence of puzzles with one evolving program. Each attempt at stage N is tested against all N puzzles solved so far, not just the newest one, so a fix that regresses an earlier puzzle is caught and reported as a regression rather than accepted.

## Sandboxing and Security

- Generated programs run as a separate subprocess with `env={}`, so they cannot read `GOOGLE_GENAI_API_KEY` or any other environment variable.
- Each run has a fixed timeout (30s); an expired run is reported as a timeout failure, not a crash.
- The API key lives only in the backend's environment, never in frontend code or generated programs.
- No sandboxing beyond process isolation and environment stripping, no filesystem restriction, no network restriction beyond what "no imports beyond the standard library" achieves by the prompt's instruction. This is a known gap; see Limitations.

## Evaluation Methods

A generated program is judged solely by whether replaying its output through the real simulator reaches a state where every movable with a `goal_position` is on it. No partial credit, No static analysis of the code. Correctness is behavioral.

## Known Limitations

- No authentication or rate limiting on any endpoint; every route, including the LLM one, can be called directly by anyone with the URL, not just through the frontend, and each LLM call has real API cost.
- No handling for prompts exceeding the model's context window; large puzzles or long curriculum chains could hit this untested.
- Mobile layout is untested; the added solver panels (.player, .solver_output) were built and verified on desktop only and may not respect the site's existing mobile breakpoints.
- Solver buttons are not disabled while a run is in progress; starting a second run before the first completes is possible and untested, and could produce corrupted/incoherent output in the log.
- No enforcement preventing a generated program from writing files or opening network connections beyond what 'stdlib only' in the prompt discourages, nothing below the prompt-instruction layer actually stops it.
- No program-length ranking metric implemented; the anti-hardcoding requirement is enforced only by curriculum's multi-puzzle re-testing, not by comparing program length across attempts.
- The harness implements curriculum mode but not a separate 'evaluate one program against several preselected puzzles in a single pass' mode without staging.