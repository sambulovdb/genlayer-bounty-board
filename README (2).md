# BountyBoard — bounties judged by GenLayer consensus

An Intelligent Contract for [GenLayer](https://genlayer.com) that lets anyone post a task with **acceptance criteria written in plain language**, and lets workers claim it by submitting a **public URL as evidence**. Whether the work is accepted is decided by the network: every validator fetches the evidence itself, asks its own LLM to check it against the criteria, and the validators must agree on the decision.

No trusted reviewer, no oracle, no off-chain judge. This is the kind of subjective, web-dependent decision a normal smart contract cannot make.

## Why this is a GenLayer use case

On a regular chain, "did this person actually write the tutorial / publish the report / fix the docs page?" needs a trusted human or an oracle. Here the contract itself:

1. fetches the evidence page (`gl.nondet.web.render`),
2. evaluates it against the poster's criteria with an LLM (`gl.nondet.exec_prompt`),
3. reaches agreement through a **custom validator** (`gl.vm.run_nondet`).

Typical uses: content and documentation bounties, grant milestones, community quests ("publish a thread that explains X"), bug-report triage.

## How consensus is designed

The interesting part is what the validators agree on.

| Step | Leader | Each validator |
|------|--------|----------------|
| Fetch | renders the evidence URL as text | renders the same URL independently |
| Judge | asks its LLM for a JSON verdict | asks **its own** LLM the same question |
| Normalise | `_clean_verdict`: strict schema, `accepted`/`rejected` only | same function |
| Compare | — | votes **yes** only if the leader's result is well-formed **and** the verdict matches its own |

Design choices:

- **Agree on the decision, not the wording.** LLMs phrase explanations differently, so `strict_eq` on the full output would fail constantly. The validator compares only the `verdict`; the leader's `reason` and `unmet_criteria` are stored as the explanation.
- **Schema checks on the leader.** A leader result that is malformed, or that says `accepted` while listing unmet criteria, is voted down (`_is_well_formed`). The same contradiction is also downgraded to `rejected` during normalisation, so an inconsistent model answer can never complete a bounty.
- **Errors must match too.** If the leader cannot fetch the page or gets unusable output, it raises a `UserError`. Validators re-run the same code, and `run_nondet` only agrees if they fail with the same message, so a leader cannot fake a fetch failure to block a valid submission.
- **Prompt-injection hardening.** The evidence page is attacker-controlled. It is truncated, fenced inside `<evidence>` tags, closing tags inside the page are neutralised, and the model is told to treat the content as data and ignore any instructions in it. Acceptance requires *every* criterion to be verifiable from the evidence.

## Contract API

### Write methods

| Method | Description |
|--------|-------------|
| `create_bounty(title, criteria, reward) -> int` | Opens a bounty. Title 1–120 chars, criteria 10–2000 chars, reward 1–1,000,000 reputation points. Returns the bounty id. |
| `submit_work(bounty_id, evidence_url) -> dict` | Submits an `https://` URL. Runs the consensus evaluation and returns `{submission_id, verdict, reason, unmet_criteria}`. On acceptance the bounty is completed and the worker is credited the reward. |
| `cancel_bounty(bounty_id)` | Poster only, while the bounty is open. |

### View methods

`get_bounty(id)`, `get_open_bounties()`, `get_submission(id)`, `get_bounty_submissions(bounty_id)`, `get_reputation(address)`, `get_attempts(bounty_id, address)`, `get_stats()`.

### Rules enforced on-chain

- The poster cannot submit to their own bounty.
- At most **3 attempts per worker per bounty**, to limit spam and evaluation cost.
- Only the first accepted submission wins; the bounty then closes.
- Cancelled or completed bounties reject new submissions.

### State

- `bounties: TreeMap[u256, Bounty]` and `submissions: TreeMap[u256, Submission]` (storage dataclasses)
- `bounty_submissions: TreeMap[u256, DynArray[u256]]` — per-bounty index, so listing a bounty's submissions does not scan everything
- `attempts: TreeMap[str, u256]` — keyed by `"<bounty_id>:<worker>"`
- `reputation: TreeMap[Address, u256]`

Rewards are reputation points rather than tokens, because GenLayer Studio does not support value transfers yet. Swapping them for a payable escrow is a contained change in `create_bounty` / `submit_work`.

## Tests

`test_bounty_board.py` runs the contract with [genlayer-test](https://pypi.org/project/genlayer-test/)'s direct mode against the real GenLayer Python SDK (runner `py-genlayer:1jb45…`, the one used by Studio's GenVM v0.2.16), with web pages and LLM answers mocked.

```bash
pip install genlayer-test
pytest -v
```

21 tests cover:

- bounty creation and input validation;
- accepted / rejected flows, reputation credit, per-bounty submission history;
- LLM output wrapped in markdown, garbage output, and self-contradictory verdicts;
- prompt-injection fencing of the evidence page;
- guards: poster self-submission, non-https URLs, unknown bounty, attempt limit, completed and cancelled bounties;
- **validator behaviour**: agrees with a leader that reached the same verdict in different words, rejects a leader with a different verdict, rejects malformed or contradictory leader results.

## Live run on GenLayer Studio

Deployed on GenLayer Studio (GenVM v0.2.16, Normal / Full Consensus). All transactions reached consensus and were finalized.

| Step | Account | Result |
|------|---------|--------|
| Deploy | poster | FINALIZED |
| `create_bounty("Reserved example domain", "The page must state that this domain is for use in documentation examples.", 50)` | poster | bounty `0` open |
| `submit_work(0, "https://example.com")` | worker | **accepted** → bounty `0` `completed`, `winner` = worker, `submission_count` = 1 |
| `create_bounty("Python tutorial", "The page must contain a Python code example.", 30)` | poster | bounty `1` open |
| `submit_work(1, "https://example.com")` | worker | **rejected** |

Validator explanation stored for the rejected submission:

> "The provided evidence is a generic example.com informational page and does not show any Python code example. Therefore the acceptance criterion is not satisfied."
>
> `unmet_criteria`: "The page must contain a Python code example."

The same URL is accepted for one bounty and rejected for another, which is the point: the decision comes from checking the live page against each bounty's own criteria.

## Try it in GenLayer Studio

1. Open [studio.genlayer.com](https://studio.genlayer.com), create a contract and paste `bounty_board.py`. If your Studio uses a different GenVM version, copy the two header lines from the built-in `storage.py`.
2. Deploy (no constructor arguments).
3. `create_bounty` with:
   - title: `Reserved example domain`
   - criteria: `The page must state that this domain is for use in documentation examples.`
   - reward: `50`
4. Switch to a different account and call `submit_work(0, "https://example.com")` → expected `accepted`.
5. Create a second bounty whose criteria require something the page does not contain, for example `The page must contain a Python code example.`, and submit the same URL → expected `rejected` with the unmet criterion listed.

## Limitations and next steps

- Evidence is limited to the first 8,000 characters of the rendered page.
- The validator compares verdicts only. For high-value bounties a stricter variant could also require agreement on the set of unmet criteria.
- Possible extensions: payable escrow, a poster appeal window, multiple evidence URLs per submission, and an allow-list of evidence domains per bounty.
