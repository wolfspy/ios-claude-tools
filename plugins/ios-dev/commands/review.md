---
allowed-tools: Bash(gh issue view:*), Bash(gh search:*), Bash(gh issue list:*), Bash(gh pr diff:*), Bash(gh pr view:*), Bash(gh pr list:*), Bash(git log:*), Bash(git blame:*), Bash(git diff:*), Bash(git merge-base:*), Bash(git rev-parse:*), Bash(git branch:*), Bash(git status:*), Bash(git symbolic-ref:*), Bash(find:*)
description: Code review a pull request or local branch diff (no GitHub comment)
disable-model-invocation: false
---

Provide a code review for the change under review, output as a chat message in this conversation. Do NOT post the review back to GitHub — this is a local-only review for the user to read.

**Identifying the change under review.** The change can come from a GitHub pull request (**PR mode**) or, when the branch has no PR yet, from the local branch diff (**local-diff mode**). Resolve the mode first, before anything else, and remember which mode you are in — later steps branch on it:

1. **If `$ARGUMENTS` contains a PR number, URL, or branch name with an open PR:** use it to identify the PR (`gh pr view <arg>` / `gh pr diff <arg>`). This is **PR mode**.
2. **If `$ARGUMENTS` is empty:** run `gh pr view` (no arguments) for the current branch.
   - If it succeeds, you are in **PR mode** — use `gh pr view` / `gh pr diff` for metadata and the diff.
   - If it fails (exit non-zero, e.g. `no pull requests found for branch "…"`, or there is no remote at all), fall back to **local-diff mode**. Do **not** treat this as an error — it is the expected path for an unpushed branch.

   **Local-diff mode** — compute the diff against the branch's base, using only the allowed read-only git commands:
   - Determine the base branch: try `git rev-parse --abbrev-ref origin/HEAD` (gives e.g. `origin/main`); if that fails, fall back to `origin/main`, then `origin/master`, then local `main`/`master`. Pick the first that exists. If `$ARGUMENTS` names an explicit base (e.g. a branch name), use that instead.
   - The change under review is the committed diff of the current branch vs that base: `git diff <base>...HEAD` (three-dot — diffs against the merge-base, so it excludes commits the base added since you branched). Use `git diff --name-only <base>...HEAD` for the changed-file list.
   - Also fold in any **uncommitted** working-tree changes if present: `git status --porcelain` to detect them, `git diff HEAD` for unstaged+staged content. Mention in the output if the review includes uncommitted changes.
   - If the branch has no commits beyond the base **and** no uncommitted changes, there is nothing to review — say so and stop.

Throughout the rest of this command, "the PR" and "the change" both refer to whichever source the mode resolved to. Wherever a step is PR-only (querying PR state, PR comments), it explicitly says what to do in local-diff mode. Treat "changed lines / changed files" as coming from the resolved diff in either mode.

To do this, follow these steps precisely:

1. Use a Haiku agent to decide whether to proceed. **In PR mode:** check if the pull request (a) is closed, (b) is a draft, or (c) does not need a code review (e.g. because it is an automated pull request, or is very simple and obviously ok). If so, do not proceed. **In local-diff mode:** there is no PR state to check, so skip the closed/draft checks entirely; only apply the "(c) trivial / obviously fine" judgement against the diff itself. A missing PR is never itself a reason to stop.

2. Use another Haiku agent to give you a list of file paths to (but not the contents of) any **relevant guidance files** from the codebase. This includes:
   - The root `CLAUDE.md` (if one exists), and any `CLAUDE.md` files in the directories whose files the pull request modified
   - Any Swift/iOS guidance skills installed for `swiftui-pro`, `swift-testing-pro`, `swift-concurrency-pro` (or any file matching `*swiftui*`, `*swift-testing*`, `*swift-concurrency*`). A skill can arrive by two different install mechanisms, so check **all** of these locations — finding the skill's `SKILL.md` in *any* of them counts as installed:
     - **Project skills:** `.claude/skills/<name>/SKILL.md`
     - **Personal skills:** `~/.claude/skills/<name>/SKILL.md` (this includes symlinks, e.g. from `npx skills add` which links `~/.agents/skills/<name>` into here — resolve symlinks when reading)
     - **Plugin cache** (how `/plugin install <name>@ios-tools` lands them): `~/.claude/plugins/cache/*/*/*/skills/<name>/SKILL.md` — i.e. `~/.claude/plugins/cache/<marketplace>/<plugin>/<version>/skills/<name>/SKILL.md`. A quick way to find them: `find ~/.claude/plugins/cache -ipath '*skills/swiftui-pro/SKILL.md'` (repeat per skill name, or glob all three). These are NOT symlinked into `~/.claude/skills/`, so they will be missed unless this path is scanned explicitly.
     Record the absolute path of whichever `SKILL.md` is found first; that path is what Agents #8–#10 read in step 4. If the same skill exists in more than one location, prefer the project skill, then the personal skill, then the plugin cache. Within the plugin cache, if a skill exists at more than one `<version>` directory, use the highest version — a lower version is a stale cache entry.
   - The project's SwiftLint configuration (`.swiftlint.yml` at the repo root or in modified directories), if one exists. Record which of Agent #1's baseline conventions the lint config already enforces (e.g. `trailing_comma` with `mandatory_comma: true` covers the trailing-comma rule, `identifier_name` covers single-letter names) so Agent #1 can skip those — lint/CI catches them on its own.
   - Detect which Apple platform layers the PR touches by inspecting the changed files' *content*, not their directory names (feature-folder layouts make path heuristics unreliable):
     - `.swift` files containing `import SwiftUI` → note SwiftUI involvement
     - `.swift` files containing `import Testing` or `import XCTest` → note testing involvement
     - `.swift` files containing `async`/`await`, `actor`, `Task {`, or `@MainActor` → note async/concurrency involvement
     - `.xcstrings`, `.xcassets`, `.plist`, `.xcconfig`, `.xib` changes → note by type
   Collect everything into a single list of paths grouped by purpose (CLAUDE.md vs Swift/iOS skills), and pass it to the review agents in step 4. For each of the three Swift/iOS pro skills, record two facts: (a) is it installed (and the absolute path to its `SKILL.md`), and (b) does the PR touch its domain. These two facts together decide whether the matching domain-specialist agent (Agents #8–#10) runs in step 4 — a specialist runs only when its skill is installed **and** the PR touches its domain. The skills are optional: it is normal for none, some, or all of them to be installed, and a missing skill is never an error.

3. Use a Haiku agent to return a summary of the change. **In PR mode:** view the pull request (`gh pr view` / `gh pr diff`) and summarize it. **In local-diff mode:** there is no PR description, so summarize from the diff itself plus the branch's commit messages (`git log <base>..HEAD --oneline`) — derive the intent from the commits and changed files.

4. Then, launch the parallel review agents below to independently code review the change. Run them on the session's model (`model: inherit`) — do not downgrade them to a smaller model to save cost; review quality tracks the model doing the reviewing. The 7 core agents always run, plus up to 3 domain-specialist agents (#8–#10) conditional on step 2's findings. Each agent should return a list of issues, each with its `file:line`, a one-line description, and the rule/reason it was flagged (e.g. CLAUDE.md rule, bug, historical context, prior PR comment, skill rule, etc.):

   a. **Agent #1 — Convention compliance:** Audit the changes for adherence to CLAUDE.md and any discovered guidance files. Check each of the following baseline Swift/iOS conventions against changed lines only — treat them as a starting point and defer to the target project's `CLAUDE.md` wherever it states its own rules (any type, wrapper, or layer names below are illustrative, not prescriptive):
      - **Naming:** protocols get the clean plain noun — no `Any` prefix (a legacy convention that collides with the `any` keyword) and no `Protocol` suffix; concrete types carry a descriptive prefix (`Live` for the production implementation, `Mock` for tests, or a meaningful characteristic like `Cached`/`InMemory`) and no `Implementation` suffix; no single-letter variable names; no file header comments (`//  FileName.swift`, copyright, etc.)
      - **Annotation placement:** property wrappers on the same line as the declaration; `@MainActor`, `@Suite`, `@Test` on the previous line, never inline with the declaration
      - **SwiftUI deprecated APIs:** `foregroundColor` (use `foregroundStyle`), `cornerRadius` (use `clipShape(.rect(cornerRadius:))`), `tabItem` (use `Tab` inside `TabView`), `NavigationView` (use `NavigationStack`), `fontWeight(.bold)` (use `bold()`), `GeometryReader` when `containerRelativeFrame()` or `visualEffect()` would work, `ScrollViewReader` (use `ScrollPosition`/`defaultScrollAnchor`), `UIGraphicsImageRenderer` (use `ImageRenderer`), `showsIndicators:` on scroll views (use `.scrollIndicators(.hidden)`), `onChange` 1-parameter variant (use 2-parameter or zero-parameter closure variant — note: in Swift anonymous closure syntax, `$0` is the first param and `$1` is the second, so a closure using `$1` has two parameters and is already the modern form; only closures using `$0` alone or a single named param are the deprecated 1-parameter variant), `onTapGesture` when `Button` would work, images as button labels without visible text
      - **SwiftUI layout:** no hard-coded padding or stack spacing values without explicit design requirement; no `AnyView` unless absolutely necessary; no `UIScreen.main.bounds`; no computed property views (extract to `View` structs); `ForEach` over `enumerated()` must match the deployment target — below iOS 26 the sequence **must** be wrapped as `ForEach(Array(items.enumerated()), id: \.element.id)`, because the unwrapped `ForEach(items.enumerated(), ...)` form requires `EnumeratedSequence: RandomAccessCollection` (iOS 26+) and is a compile error under the Swift 6 language mode; only flag the `Array(...)` wrapper as unnecessary when the project's deployment target is iOS 26 or later
      - **Observable state:** new shared data should use `@Observable` + `@State`/`@Bindable`, not `ObservableObject`/`@Published`/`@StateObject`/`@ObservedObject`; any `@Observable` class must be `@MainActor`
      - **Logging:** new non-trivial functions should include entry/exit trace logging; route all logging through the project's logging abstraction — never `print()` or `os_log`; log-and-abort on handled errors that cancel an operation, log-and-continue on non-fatal guard short-circuits; tag log entries with the relevant entity identifiers per the project's convention
      - **Testing:** Swift Testing `@Test`/`#expect`/`#require` — not XCTest; suites as `struct` not `class`; backtick natural-language test names; no `testXxx` prefix; group members by visibility (lifecycle → internal → private); `@Suite(.serialized)` only when shared mutable state; no nested `@Suite`s; follow the project's mock-naming convention
      - **Accessibility identifiers:** all new interactable SwiftUI elements (buttons, text fields, toggles, pickers) should have a stable `.accessibilityIdentifier(_:)` for UI testing, following the project's naming convention
      - **Colors:** use the project's existing color palette / design tokens; no ad-hoc new colors; no UIKit colors in SwiftUI code
      - **Concurrency:** no `DispatchQueue.main.async` (use `await MainActor.run` or `@MainActor`); no `Task.sleep(nanoseconds:)` (use `Task.sleep(for:)`); strict Swift 6 concurrency assumed
      - **Formatting:** no `String(format:)` for numbers (use `FormatStyle`); no `DateFormatter`/`NumberFormatter`/`MeasurementFormatter` (use modern `FormatStyle` API); trailing commas on last element in multi-line argument lists, array literals, and tuple literals
      - **Localization:** new user-visible strings added to `.xcstrings` with all languages translated; natural-language keys not dot-notation identifiers
      - **Feature flags:** read flags through the project's feature-flag abstraction, not raw inline flag checks
      - **Networking:** new API calls should follow the project's established networking layering (endpoint definition → request building → typed service → execution); prefer `async`/`await` over completion handlers
      Note: Apply rules to changed lines only — do not flag pre-existing violations. Skip any baseline rule that step 2 found is already enforced by the project's SwiftLint configuration — lint/CI catches those, and step 5's verification would drop them anyway; checking them here is wasted work. When the PR touches SwiftUI, tests, or concurrency, leave the deep analysis of those domains to the specialist agents (#8–#10) and focus here on CLAUDE.md and project-convention conformance.

   b. **Agent #2 — Bug scan:** Read the file changes and do a focused scan for Swift/iOS bugs. Read the surrounding context needed to judge the changes properly — the full enclosing file, plus the direct call sites of any changed function or protocol requirement (grep for the symbol) — because retain cycles, actor-isolation violations, and broken call-site assumptions are often invisible in the diff alone. Don't wander beyond that (no repo-wide exploration). Focus on large, impactful bugs, not nitpicks or style issues; step 5's verification pass absorbs false positives, so prefer recall over caution. Examples to look for:
      - Retain cycles in closures (missing `[weak self]` or `[unowned self]`); `unowned` used where the captured object could be deallocated before the closure fires
      - Force unwraps (`!`) or force `try!` on values that can realistically be nil or throw at runtime
      - Actor isolation violations: accessing `@MainActor`-isolated state from a non-isolated context, or crossing actor boundaries without `await`
      - Fire-and-forget `Task { }` where cancellation matters and no handle is stored
      - CoreData objects accessed or faulted on a background thread without a proper `perform`/`performAndWait` block
      - Missing `await` on async calls, or `async`/`throws` not propagated correctly through call chains
      - Incorrect `Sendable` conformance or `@unchecked Sendable` used to silence a real data-race
      - Logic errors in `guard`/`if let`/`switch` chains that silently skip a code path instead of surfacing an error
      - Incorrect `weak` reference to a value type (structs/enums cannot be `weak`)
      - Missing error propagation — caught errors that are silently swallowed with no logging or recovery
      Ignore changes in behavior that are clearly intentional given the PR's stated goal.

   c. **Agent #3 — Historical context:** Read the git blame and history (`git log -p`, `git blame`) of the code modified, to identify any bugs or regressions in light of that historical context. Look for: reversions of intentional decisions, changes that conflict with comments explaining prior design choices, patterns that were previously fixed being reintroduced.

   d. **Agent #4 — Prior PR comments:** Read previous pull requests that touched these files using `gh pr list` and `gh pr view`, and check for any comments on those PRs that may also apply to the current change. Flag anything that was previously called out as a problem and appears again. This agent queries the repo's *historical* PRs, so it works in both modes (the current branch needn't have its own PR). If `gh` returns nothing or fails because there is no remote (a purely local repo), skip this agent gracefully — it is not an error, just return no findings.

   e. **Agent #5 — Code comment compliance:** Read the code comments and inline documentation in the modified files. Make sure the changes comply with any guidance, constraints, or intent expressed in those comments. Also flag any newly added comments that describe *what* code does rather than *why* (CLAUDE.md says: no explanatory comments about what the code does).

   f. **Agent #6 — Silent-failure audit:** Examine every error-handling site the change adds or modifies and flag failures that would vanish without a trace. Language-level baseline (applies to any Swift project):
      - Empty `catch` blocks, or `catch` blocks that neither log, rethrow, nor surface the failure to the caller or user
      - `try?` that discards an error where the caller has no other way to learn the operation failed
      - Broad `catch` clauses that swallow unrelated errors alongside the expected one
      - `guard`/`if let` early returns on failure paths with no logging and no propagation — the operation just silently stops
      - Fallback values or fallback code paths taken on error with no record that the primary path failed
      - Retry loops that exhaust their attempts and continue as if nothing happened
      - Optional chaining used to silently skip operations that can meaningfully fail
      If the project's CLAUDE.md (or another guidance file from step 2) defines a logging abstraction or an error-handling policy (e.g. which log level a handled-and-aborted error must use), check each site against that policy specifically; otherwise apply the baseline above. Deliberate, visibly-acknowledged suppression (e.g. a `// best-effort, failure is fine` comment) is not a finding.

   g. **Agent #7 — Test-coverage gaps:** Judge whether the change's *new or modified logic* has behavioral test coverage — tests that would fail if that logic regressed. This is about coverage of the change, not the repo's overall coverage. Look for:
      - New logic branches (conditionals, error paths, state transitions) that no test exercises
      - Bug fixes with no regression test reproducing the original bug
      - Tests that mirror the implementation so closely they cannot fail when behavior changes (asserting what the code *does* rather than what it *should do*)
      - Edge cases the diff visibly introduces (empty collections, nil, boundary values) that no test touches
      Rate each gap by impact: untested logic on a critical path (persistence, data loss, money, auth) outranks a missed cosmetic edge case. Defer to the project's stated testing policy in CLAUDE.md when it has one (e.g. "tests encouraged but not mandatory" lowers severity; "all new logic must be tested" raises it); when the project states none, flag only the gaps a senior engineer would insist on. Do not demand tests for trivial pass-through code, pure UI layout, or generated code.

   The next three are **optional domain specialists**. Each one runs the corresponding installed skill's *own* review process against the changed files — so you get the full depth of `/swiftui-pro`, `/swift-testing-pro`, and `/swift-concurrency-pro` inside this single `/review` run, without invoking them separately. **Spawn a specialist only when step 2 reported that its skill is installed AND the PR touches its domain.** If either is false, skip that agent entirely (do not spawn it). These skills are not required: if none are installed, `/review` runs normally with just the 7 core agents above. Never treat a missing skill as an error or block the review on it — silently omit the corresponding specialist and continue. (The one exception is the optional install nudge in step 7: when the PR touched a domain but its skill wasn't installed, the review may surface a single suggestion at the end — but it is never an error and never interrupts the run.) Each specialist that does run carries the skill's own priority label — `(high)`, `(medium)`, or `(low)` from its prioritized summary — through with every finding, so step-5 verification can map it straight to severity.

   h. **Agent #8 — SwiftUI deep review** *(only if `swiftui-pro` is installed and the PR touches SwiftUI):* Read the skill's `SKILL.md` (path from step 2) and follow its review process step by step, loading the `references/` files it points to as needed (api, views, data, navigation, design, accessibility, performance, swift, hygiene). Apply it to the changed SwiftUI files only, on changed lines. Return issues in the same `file:line` + one-line description + rule/reason format, citing the specific skill reference (e.g. `swiftui-pro/references/performance.md`) as the reason.

   i. **Agent #9 — Swift Testing deep review** *(only if `swift-testing-pro` is installed and the PR adds or changes tests):* Read the skill's `SKILL.md` and follow its review process, loading its `references/` files as needed (core-rules, writing-better-tests, async-tests, new-features; migrating-from-xctest only if the PR converts XCTest). Apply it to the changed test files only. Same output format, citing the specific skill reference as the reason.

   j. **Agent #10 — Swift concurrency deep review** *(only if `swift-concurrency-pro` is installed and the PR touches async/concurrency code):* Read the skill's `SKILL.md` and follow its review process, loading its `references/` files as needed (hotspots, actors, structured, unstructured, cancellation, async-streams, bridging, bug-patterns, diagnostics). Apply it to the changed concurrency-relevant files only. Same output format, citing the specific skill reference as the reason.

5. **Two-stage verification.**

   **5a — Dedup and triage (single Haiku agent).** Combine the issue lists from all step-4 agents into one list and hand it, with the diff under review (PR diff or local branch diff), to a single agent. This stage genuinely needs the whole list at once — that is what makes dedup possible — but it should only cut the obvious; real judgement happens per-issue in 5b. It must:
   - Drop duplicates and merge near-duplicates raised by different step-4 agents into one issue.
   - Drop issues that are not on a line the change actually modified (pre-existing code).
   - Drop obvious nitpicks a senior iOS engineer wouldn't call out in a real PR review, and anything a Swift compiler, SwiftLint, or CI would catch on its own.

   **5b — Per-issue verification (one agent per surviving issue, launched in parallel, session model).** For each issue that survives 5a, launch an independent verifier. Each verifier receives the diff, the step-2 guidance-file paths, and *only its own issue* — judging one issue per context keeps attention undiluted and stops earlier verdicts anchoring later ones. Each verifier must:
   - Open the flagged file at the flagged lines and confirm the problem is really there in context, not just plausible from the diff hunk.
   - Verify any cited CLAUDE.md or guidance-file rule actually says what the review agent claimed — open the file, find the line, and quote it. For domain-specialist findings (Agents #8–#10), the cited rule lives in the skill's reference files (e.g. `swiftui-pro/references/performance.md`); verify those the same way.
   - Actively try to refute the issue: look for the reason the code is actually fine (intentional given the change's stated goal, guarded elsewhere, unreachable in practice). An issue the verifier genuinely cannot refute earns high confidence; one it cannot confirm stays low.

   Each verifier returns its issue's `file:line`, one-line description, the rule/reason, and:
   - `confidence` (0–100): how sure is the agent this is real?
   - `severity`: `high`, `medium`, or `low`
   - `type`: `Bug` | `Style` | `Testing` | `Logic`

   Confidence rubric (give this verbatim to each verifier):
   - **0:** False positive, or pre-existing.
   - **25:** Might be real, couldn't verify. Stylistic issues not explicitly in CLAUDE.md land here.
   - **50:** Verified real, but might be a nitpick or rare in practice.
   - **75:** Double-checked, likely hit in practice, important, or directly named in CLAUDE.md.
   - **100:** Confirmed real, will happen frequently, evidence directly confirms.

   Severity rubric (give this verbatim to each verifier):
   - **`high`:** breaks functionality, causes incorrect behavior, security issue, data loss, a silent failure that swallows errors without logging or surfacing them, OR a domain-specialist finding the skill ranked `(high)`.
   - **`medium`:** violates a CLAUDE.md or project-convention rule, a meaningful maintainability/refactor issue, a behavioral test-coverage gap on important logic, OR a domain-specialist finding the skill ranked `(medium)`.
   - **`low`:** minor style, readability, small cleanups, missing tests on non-critical paths, OR a domain-specialist finding the skill ranked `(low)`.
   For domain-specialist findings, the skill's own priority is authoritative — map its `(high)`/`(medium)`/`(low)` label straight to the matching severity.

6. Drop any issue with `confidence < 75`. (The rubric's 75 band explicitly covers rules directly named in CLAUDE.md — those findings must survive the cut.)

7. Output the review as a chat message directly in the conversation (do NOT post to GitHub). Keep output brief. Use the following emojis to prefix each issue line for quick visual scanning: 🔴 for high severity, 🟡 for medium severity, ⚪ for low severity, 🐛 for Bug type, 🎨 for Style type, 🧪 for Testing type, 🔀 for Logic type — combine both the severity and type emoji on each issue (e.g. `🔴 🐛`). Tag each issue with its `[severity · type]` label as well, and link/cite relevant code so the user can navigate to it.

   **Optional install nudge.** After the issue list, check step 2's findings for any domain that the PR touched but whose specialist skill was *not* installed. If there is at least one, add a `💡` tip at the very end suggesting the user install those specific skills for deeper review next time, and include the **exact copy-pasteable install command(s)** so the user doesn't have to look them up. The command for each skill is `/plugin install <name>@ios-tools` (e.g. `/plugin install swiftui-pro@ios-tools`). Put each command on its own line in a fenced code block so it's easy to copy. Name only the skills that were both relevant (PR touched the domain) and missing — never mention a skill that is already installed, and never nudge about a domain the PR didn't touch. If every relevant skill was installed (or the PR touched none of the three domains), omit the nudge entirely. The nudge is informational only — it is not a finding, does not count toward the issue total, and never appears mid-review. The three install commands are exactly: `/plugin install swiftui-pro@ios-tools`, `/plugin install swift-testing-pro@ios-tools`, `/plugin install swift-concurrency-pro@ios-tools`. Example:

   ````
   💡 This change touched SwiftUI and tests — install these for deeper domain review next time:

   ```
   /plugin install swiftui-pro@ios-tools
   /plugin install swift-testing-pro@ios-tools
   ```
   ````

---

## False positives — examples for steps 4 and 5

- Pre-existing issues not touched by the PR
- Something that looks like a bug but is actually intentional or safe in context
- Pedantic nitpicks a senior iOS engineer wouldn't call out
- Issues a compiler, Swift linter (SwiftLint), or type checker would catch — do not run these yourself; CI handles them
- General security or documentation-quality issues unless explicitly required in CLAUDE.md (behavioral test-coverage gaps in the *changed* logic are in scope — Agent #7 exists to find them)
- Issues silenced with a `// swiftlint:disable` comment or similar
- Functionality changes that are clearly intentional and directly related to the PR's stated goal
- Real issues on lines the author did not modify

---

## Notes

- Do not attempt to build or compile the app. Build and lint run separately in CI.
- Use `gh` read-only to fetch PR metadata and diffs in PR mode. Do NOT use `gh pr comment` or any write commands.
- In local-diff mode, use only read-only git commands (`git diff`, `git merge-base`, `git rev-parse`, `git log`, `git blame`, `git status`, `git branch`) to source the change. Never push, commit, or otherwise mutate the repo — `/review` is read-only.
- Create a todo list before starting.
- Cite and link every issue in the final output.

---

## Output format

If issues were found (example with 3):

---

### Code review

Found 3 issues:

1. 🔴 🐛 **Brief description** `[high · Bug]` (explanation of why it breaks)

   `path/to/File.swift:12-15`

2. 🟡 🎨 **Brief description** `[medium · Style]` (CLAUDE.md says "…exact quoted rule…")

   `path/to/File.swift:42-46`

3. ⚪ 🔀 **Brief description** `[low · Logic]` (`swiftui-pro/references/data.md`, ranked low)

   `path/to/File.swift:88`

💡 This change touched SwiftUI — install it for deeper domain review next time:

```
/plugin install swiftui-pro@ios-tools
```

---

(The `💡` nudge appears only when the change touched a domain whose specialist skill wasn't installed; otherwise omit it. List one `/plugin install …@ios-tools` line per missing-but-relevant skill.)

If no issues were found:

---

### Code review

No issues found. Checked for bugs and CLAUDE.md compliance.

💡 This change touched concurrency code — install it for deeper domain review next time:

```
/plugin install swift-concurrency-pro@ios-tools
```

---

(Same rule for the no-issues case: include the `💡` nudge only if a relevant specialist skill was missing.)

For code links, prefer a local-style reference: `path/to/File.swift:Lstart-Lend` (clickable in terminals/editors). Alternatively provide a full GitHub URL with the commit SHA: `https://github.com/owner/repo/blob/<full-sha>/path/to/File.swift#Lstart-Lend`. Include at least 1 line of context before and after the flagged line.
