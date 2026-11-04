# UnvibeCode challenge: findings and adoption proposal

Agent-assisted public-repository trials for the UnvibeCode Engineering Challenge 2026.

## Status

- Track 2 was submitted through the challenge form on 4 November 2026.
- Track 1 below is a prepared issue draft, not an issue posted to the upstream tracker.
- AI assistance is disclosed. These are tool-run observations, not a claim that the participant personally executed every command.
- Tested with unvibecode 0.3.15 on Linux / Python 3.12. Ruby reviews were inspected at the completed code-understanding stage using interim context ZIPs; they are not claimed as completed full workflow reviews.

## Repositories tested

- Flask: https://github.com/pallets/flask
- Jekyll: https://github.com/jekyll/jekyll at `97c3bdc822df9d5bea4c269b29a28de5a5e29d70`
- RSpec Core: https://github.com/rspec/rspec-core at `aec5f49485d97908183dbe790a7fefb8baaa8091`

## Reproduce the Ruby finding

1. Install unvibecode 0.3.15 with Python 3.11 or later.
2. Clone the public repositories and check out the pinned commits above.
3. Run `python -m unvibecode review --repository /path/to/jekyll`.
4. When the code-understanding stage is complete, extract the repository-context ZIP.
5. Inspect `unresolved_relationships.jsonl`, `connections.jsonl`, `chunks.jsonl` and `selections.jsonl` using the examples in the issue draft below.

No raw source archives, account data, credentials or private code are included. Output excerpts below are short extracts from analysis of public repositories. Counts are observations from the tested checkouts, not guarantees for future versions. Repeat tests matched in these runs; universal determinism was not established.

## Track 2: submitted adoption proposal

Recommended approach - "UnvibeCode Watch": a GitHub Action that re-reviews every pull request and posts a branded workflow-and-risk delta comment.

Turn UnvibeCode from a one-time review into a recurring part of the pull-request loop: a CI step that compares the connected-code graph and business workflows between the base and head commits and comments "what changed" on every PR, linking to the full branded report.

Why this drives repeat usage (supporting points):

1. It attaches to a trigger developers already have daily. Developers open and review pull requests every day; nobody opens a codebase-analysis tool daily out of habit. In an agent-assisted trial of the product prepared for me, the value ended in a single session: one review, one downloaded ZIP, and after that no reason to come back - the product's name leaves the workflow the moment the export does. A PR delta comment arrives exactly when the question "what does this change touch?" is live, so repeat usage rides an existing habit instead of demanding a new one.

2. It is feasible with today's engine. In that trial, the code-understanding lane was deterministic across repeat runs: two reviews of the same Flask checkout produced byte-identical outputs (403 connections, 482 chunks both times), which makes commit-to-commit diffs meaningful rather than noise. That lane finished in seconds-to-minutes with no hosted analysis in these runs, so a delta comment is CI-viable; the heavier business-workflow review can run against a cached baseline or on a schedule. The product does not need a new analysis capability, only a diff-and-notify wrapper.

3. The brand shows up at the exact moment of value, unlike an invisible MCP call. The PR comment and every linked report page carry the UnvibeCode name and its "verified connections" framing at the moment a reviewer needs context. A developer may benefit from an MCP integration without ever noticing the product's name; they cannot miss the name on a comment they read and a report they open before merging. Each report view is also a natural re-entry point into running a full local review.

Implementation plan (2-3 lines):

Build an official "unvibecode watch" GitHub Action that caches the fast-lane resolved graph per base commit, diffs connections and workflows for each PR, and posts one branded summary comment ("3 workflows changed, 1 new risk finding") with a link to the full report. Test repeat usage by measuring weekly active repositories with the action installed, 7/30-day re-trigger rate per repo, and report-link opens per team against a control group of one-time local reviews.


## Track 1: prepared engineering issue draft

Title: [Bug] Connected Code Map / Ruby linker - plain `require` only probes repo root, `lib/` and `app/`, so requires from load-path directories like `test/` and `spec/` are dropped (51 of 51 in jekyll, 20 in rspec-core)

## 1. Problem summary

For Ruby repositories, UnvibeCode resolves plain `require "x"` statements by probing only three locations: the repository root, `lib/`, and `app/`. Real Ruby projects put additional directories on the load path - most commonly `test/` and `spec/` for the test harness (`require "helper"`, `require "test_helper"`, `require "spec_helper"`, `require "support/…"`). In the two repositories tested, every such require is reported `unresolved / unknown_symbol_or_external_module`, even when the target file exists in the repository and was fully analyzed.

As a result the test suite is disconnected from its own harness in the Connected Code Map, and the "connected code" download for an affected test file contains only the file itself - the helper that defines its base classes, shared configuration and support code is silently omitted from the LLM context a developer exports.

This is not an exotic layout: it is the standard Ruby test convention. Jekyll's own `Rakefile` puts `test/` on the load path (`test.libs << "lib" << "test"`), and rspec-core's `spec/spec_helper.rb` documents the same for `spec/` ("Ensure requires are relative to `spec`, which is on the load path").

## 2. Test environment

- UnvibeCode version: 0.3.15 (current PyPI release, `pip install unvibecode`)
- Operating system / Python version: Linux x86-64 (Ubuntu 22.04), Python 3.12 (tested on Linux only)
- Public repository URL and commit:
  - https://github.com/jekyll/jekyll @ `97c3bdc822df9d5bea4c269b29a28de5a5e29d70`
  - https://github.com/rspec/rspec-core @ `aec5f49485d97908183dbe790a7fefb8baaa8091`
- Relevant command: `python -m unvibecode review --repository "/path/to/jekyll"`

## 3. Steps to reproduce

1. `git clone --depth 1 https://github.com/jekyll/jekyll && cd jekyll`
2. `python -m unvibecode review --repository "/path/to/jekyll"` and wait for the Code understanding stage to finish (the defect is in the fast-lane stage, which produced identical output across repeated runs in this test; the interim `jekyll-repository-context.zip` already shows it).
3. Extract `jekyll-repository-context.zip` and inspect two files:
   - `unresolved_relationships.jsonl` - search for `"module": "helper"`
   - `selections.jsonl` - inspect the scopes of any `test/test_*.rb` file

## 4. Expected behavior

- `require "helper"` in `test/test_ansi.rb` (line 3) should resolve to `test/helper.rb`. At runtime it does: Jekyll's `Rakefile` (line 140) declares `test.libs << "lib" << "test"`, so `test/` is on Ruby's `$LOAD_PATH`, and plain `require` searches the load path. The same applies to rspec-core, where `spec/spec_helper.rb` (lines 22-24) states "Ensure requires are relative to `spec`, which is on the load path", so `require 'support/aruba_support'` loads `spec/support/aruba_support.rb`.
- The Connected Code Map should show an edge from each of the 51 `test/test_*.rb` files to `test/helper.rb`, and the radius-1 connected-code download for a test file should include the harness it depends on.

## 5. Actual behavior

- All 51 of the 51 jekyll test files that contain `require "helper"` (every `test/test_*.rb` that uses the harness) produce an unresolved record:

```json
{"detail": "unknown_symbol_or_external_module", "line": 3, "module": "helper", "raw": "helper",
 "reason": "unknown_symbol_or_external_module", "resolution_status": "unresolved",
 "source_file": "test/test_ansi.rb", "symbol_resolutions": [], "symbols": []}
```

- `test/helper.rb` is fully analyzed (it appears in `chunks.jsonl`) but receives zero inbound edges from the 51 files that require it. Only 6 connections originate from the entire `test/` tree (~80 test files).
- The precomputed selections confirm the customer-facing impact - every scope of `test/test_ansi.rb` contains only the file itself:

```text
scope 0 (radius 0): ['test/test_ansi.rb']
scope 1 (radius 1): ['test/test_ansi.rb']
scope 2 (radius 2): ['test/test_ansi.rb']
```

- Second repository, same defect: in rspec-core all 20 `require 'support/aruba_support'` / `require 'support/formatter_support'` statements across 10 `spec/integration/*_spec.rb` files are unresolved, and `connections.jsonl` contains 0 edges into `spec/support/`, although both target files are analyzed.

## 6. Evidence

- Relevant file/function names and line numbers:
  - `test/test_ansi.rb:3` → `require "helper"`; target `test/helper.rb` (exists, analyzed)
  - `jekyll/Rakefile:140` → `test.libs << "lib" << "test"` (load-path declaration)
  - `rspec-core/spec/integration/bisect_runners_spec.rb:1` → `require 'support/aruba_support'`; target `spec/support/aruba_support.rb` (exists, analyzed)
  - Shipped engine source (shipreadyv2 fast lane), `LanguageAwareImportResolver._resolve_ruby`: the plain-`require` branch builds candidates as `raw + ".rb"`, `lib/<raw>.rb`, `app/<raw>.rb`, with no probe of configured load-path roots (the requiring file's own directory is only probed in the separate `require_relative` branch).
- Output excerpts: the JSON record and selection scopes quoted in section 5 are verbatim from `unresolved_relationships.jsonl` and `selections.jsonl`.
- Source-code comparison supporting the finding: runtime truth (Rakefile load-path declaration and rspec-core's own comment) vs. the hardcoded three-location probe list above.
- Repeatable across the runs performed: two independent reviews of the same jekyll checkout produced byte-identical counts (416 unresolved records, 51 unresolved `helper` requires, 40 connections), and the same code path reproduces on rspec-core.

## 7. Impact and scope

Who is affected: any Ruby repository whose test or support files use plain `require` against a load-path directory - the standard minitest/RSpec layout (`test/`, `spec/`, `spec/support/`). Confirmed on two public repositories, jekyll and rspec-core. The effect is missing context, not a crash: in the exports inspected for these two repositories, the Connected Code Map and the exported LLM context packages for test files omit the harness and support code those files are built on (for example, every scope of jekyll's `test/test_ansi.rb` contains only the file itself), so connected-code downloads and workflow reconstruction for these repositories silently under-report the test layer. Within the two tested repositories the pattern hit every affected file (51/51 in jekyll, 20/20 in rspec-core).

## 8. Suspected cause / suggested fix (if known)

Cause: in `_resolve_ruby`, the `else` branch for plain `require` only tries `<raw>.rb` (repo root), `lib/<raw>.rb` and `app/<raw>.rb`. At runtime, plain `require` resolves through Ruby's `$LOAD_PATH`, which projects extend through their own configuration - the engine's candidate list covers none of those configured load-path roots.

Suggested fix: for plain `require`, resolve against load-path roots discovered from project configuration (`Rakefile` `test.libs`, `.rspec`, gemspec `require_paths`) before giving up. Where no configuration declares them, conventional roots such as `test/` and `spec/` could be probed as an explicitly labeled heuristic - plain `require` does not guarantee those directories are on the load path, so heuristic matches should be recorded as such and kept distinguishable from config-derived matches. An engineer can validate the fix by re-reviewing jekyll and checking that `test/test_ansi.rb` gains an edge to `test/helper.rb` and that its radius-1 selection includes `test/helper.rb`.

## 9. Duplicate check

Closest existing issues checked, and why this is distinct:

- #273 "Ruby dependency resolution misses parenthesized requires and relative parent paths" - same function, different branch and trigger: that issue covers `require(...)` parenthesized syntax (extraction) and `require_relative '../…'` parent traversal (no `..` normalization). Fixing either does not touch the plain-`require` candidate list, which has no configured load-path probe at all. Reproduced here with plain, unparenthesized `require "helper"` and `require 'support/aruba_support'`.
- #292 "Critical functional defect in Ruby/Rails connected code" - covers Rails autoloading where no `require` lines exist at all; this issue is about explicit `require` lines that exist, are extracted, and are resolved against the wrong search roots.
- #241, #378 - RSpec complexity classification and tree-sitter grammar download failure; unrelated mechanisms.
- #366 / #425 / #316 - Python import mis-resolution (file-stem fallback); different language and code path.
