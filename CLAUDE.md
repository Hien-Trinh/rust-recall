# recall

A spaced-repetition CLI in Rust that turns my markdown notes into flashcards.
This is my **first Rust project**. Learning Rust is the main goal; the tool is how I learn it.

## Coaching rules (read first, always apply)

- **I write the core logic. You do not.** Core = parser, SM-2 scheduler, FNV-1a hash, arg parsing, date math, dir walk, shuffle, review loop, store load/save.
- You MAY write: test stubs (signatures + a failing `todo!()`), README prose, CI YAML, `vhs` tape files.
- When I'm stuck, use the hint ladder. Go one step at a time and wait for me between steps:
  1. Name the Rust concept involved (e.g. "this is a borrow outliving its owner").
  2. Give a pointed hint or the relevant Rust Book / std docs section.
  3. Show a small snippet **only when I say "show me"**.
- Compiler errors: explain what the error means and why Rust enforces it. Never just paste the fix.
- Review my code when I ask: flag non-idiomatic Rust (Python-in-Rust-syntax), unnecessary `.clone()`, `unwrap()` outside tests.
- Don't add crates. Allowed dependencies: `serde` (with `derive`), `serde_json`. Everything else comes from std.

## Spec (v1)

**Card source:** plain markdown files under one root folder, walked recursively.
Deck = the name of the immediate subfolder. No tags.

```markdown
Q: What is the derivative of x^2?
A: 2x
```

- One card per `Q:`/`A:` pair. The answer runs until the next blank line or `Q:`.
- Malformed blocks become a `Result::Err` carrying file + line, then get reported and skipped. They never cause a panic.
- LaTeX is shown raw. Unicode conversion is a stretch goal.

**Card identity:** FNV-1a 64-bit hash of the normalized question (trimmed, internal whitespace collapsed).
Editing the answer keeps the card's history; editing the question makes a new card. Don't use `DefaultHasher`: it isn't stable across Rust versions.
**Never modify the note files.**

**Scheduler:** SM-2, hand-written. Card state is an `enum` (`New`, `Learning`, `Review`, `Relearning`).
Grades 1–4 (Again / Hard / Good / Easy). Time unit = whole days since the Unix epoch, computed from `SystemTime`. No `chrono`.
Shape: `fn schedule(card: &CardState, grade: Grade, today: u32) -> CardState`. Keep it pure so FSRS can be swapped in later.

**Store:** one JSON file (`~/.recall/state.json`) via `serde_json`, mapping card hash → scheduling state.

**Review loop (CLI):** show the question, wait for Enter, show the answer, read a 1–4 grade.
Due reviews come first, then up to **20 new cards/day**, shuffled with a hand-written xorshift PRNG.

**Commands** (args parsed by hand from `std::env::args`):
- `recall review [--deck NAME]`
- `recall stats` (due today, new left, total per deck)

## Milestones (30 hrs total)

| # | Hours | Goal | Done when |
|---|-------|------|-----------|
| M0 | 8 | Rust ramp | Rust Book ch. 1–10 + rustlings |
| M1 | 4 | Parser | `parse(&str) -> Vec<Result<Card, ParseError>>` with tests |
| M2 | 4 | SM-2 + FNV-1a | `schedule` tests pass; hash matches known FNV-1a vectors |
| M3 | 6 | Store + review loop | I review real cards from my notes |
| M4 | 4 | Decks + limits | dir walk, `--deck`, 20 new/day, `stats` |
| M5 | 4 | Ship | GitHub Actions: `cargo test` + `cargo clippy -- -D warnings`; README with `vhs` GIF and a "What I learned about Rust" section |

Stretch goals (only after M5): ratatui TUI · cloze `{{…}}` syntax · LaTeX→Unicode · FSRS · drop serde for a hand-written format.

## Out of scope for v1

LLM-generated cards, web UI, sync, auth, SQLite, tags, async.
