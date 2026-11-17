# Failure Modes I Hit While Shipping with a CLI Agent

*Zhong Zihao · Nov 2026 · Context: built an AI job-application assistant from scratch using a CLI agent (Claude Code / DSH) — 13 test files, 270 assertions.*

Every item below is something I actually hit and then fixed. Format: **symptom → root cause → what I did → the rule I kept.**

---

## 1. Constraints vanish after context compaction

**Symptom.** Past ~400k tokens the agent started touching things I had explicitly frozen ("never auto-commit"). The code still ran; the contract was broken.
**Root cause.** Compaction summaries keep *what was done*, not *what was forbidden*. Negative constraints are the first thing to disappear.
**Fix.** Moved every hard constraint into repo files (`AGENTS.md`) and made it read them at the start of each task.
**Rule.** A constraint that isn't in a file doesn't exist.

## 2. Parameter hallucination on tool calls

**Symptom.** Asked it to patch line 172 of `boss_judge.py`; it patched `judge.py`. It compiled. Behaviour was wrong.
**Root cause.** In long contexts the model does approximate matching on file names and line numbers instead of exact retrieval.
**Fix.** Required it to `grep` and print the candidate location, then wait for my confirmation before editing.
**Rule.** Any position-based edit must print evidence first — file, line, original text.

## 3. Silent death in long-running jobs

**Symptom.** A batch run produced no output for 74 minutes. I assumed "slow". The process had died long before.
**Root cause.** `| tail` buffered the output away, and there was no heartbeat, so liveness was unobservable from outside.
**Fix.** Log to file, one timestamped line per item; judge liveness by file mtime.
**Rule.** Anything over two minutes needs a heartbeat, and logs must go to disk.

## 4. Over-permissioned deletion

**Symptom.** "Clean up the duplicate rows" turned into deleting six legitimate records in the same directory.
**Root cause.** The model invented its own matching predicate for a destructive operation. No allow-list.
**Fix.** Destructive operations now emit a "to be deleted" list for review, and the scripts default to `--dry-run`.
**Rule.** Default to not writing. Print what you're about to write.

## 5. Concurrent writes and silent row loss

**Symptom.** Four workers ran concurrently; the CSV came back three rows short. No error anywhere.
**Root cause.** Read-modify-write wasn't atomic — each worker read the whole table and wrote it back.
**Fix.** Workers compute only; a single writer commits. Ledger became append-only JSONL; the CSV is now a derived view.
**Rule.** Under concurrency, writes must converge on exactly one writer.

## 6. Confusing action with outcome

**Symptom.** The script reported "sent", but the conversation showed no such message.
**Root cause.** Clicking a button was treated as success — "I performed the action" was confused with "the world now matches what I wanted".
**Fix.** Success is only declared when the sent content is read back from the conversation. Otherwise the state is `unknown`, and we never auto-retry.
**Rule.** Action ≠ outcome. Put the success criterion in the outside world, not in an `if`.

## 7. Metric drift

**Symptom.** The same metric scored 0.667 yesterday and 0.72 today. Both runs "passed".
**Root cause.** Thresholds, allow-lists and weights were hard-coded in several places; one got updated, another didn't.
**Fix.** Everything funnels through a single `config.py`; any change to a metric definition carries a version string into the output.
**Rule.** Metric definitions are product assets. Version them; don't trust memory.

## 8. Missing evidence treated as counter-evidence

**Symptom.** The model scored "not mentioned in the resume" as "does not have the skill" — for skills the candidate demonstrably had.
**Root cause.** Only two buckets existed (has / hasn't). There was no "I can't see evidence" bucket.
**Fix.** Added a third state, `insufficient`. A "not satisfied" verdict now requires a verbatim quote, otherwise it degrades to "undetermined".
**Rule.** Absence of evidence is not evidence of absence. Give it its own state.

---

## How I use this list

- Before any new agent task, I scan these eight and paste the relevant ones into the constraint block of the prompt.
- Every new failure gets appended here **and** turned into an executable check — an assertion in a test, not a suggestion in a doc.
- If asked in an interview what an agent's failure modes are, I lead with 1, 5 and 6 — those three cost me real time.
