---
allowed-tools: Bash(gh issue view:*), Bash(gh search:*), Bash(gh issue list:*), Bash(gh pr diff:*), Bash(gh pr view:*), Bash(gh pr list:*), Bash(git log:*), Bash(git blame:*)
description: Code review a pull request locally (no GitHub comment)
disable-model-invocation: false
---

Provide a code review for the given pull request, output as a chat message in this conversation. Do NOT post the review back to GitHub — this is a local-only review for the user to read.

**Identifying the pull request:** If `$ARGUMENTS` contains a PR number, URL, or branch name, use that to identify the PR (pass it directly to `gh pr view`). If `$ARGUMENTS` is empty, review the PR associated with the current branch (`gh pr view` with no arguments).

To do this, follow these steps precisely:

1. Use a Haiku agent to check if the pull request (a) is closed, (b) is a draft, or (c) does not need a code review (e.g. because it is an automated pull request, or is very simple and obviously ok). If so, do not proceed.

2. Use another Haiku agent to give you a list of file paths to (but not the contents of) any **relevant guidance files** from the codebase. This includes:
   - The root `CLAUDE.md` (if one exists), and any `CLAUDE.md` files in the directories whose files the pull request modified
   - Any Swift/iOS guidance skills installed in `.claude/skills/` or `~/.claude/skills/` — specifically look for `swiftui-pro`, `swift-testing-pro`, `swift-concurrency-pro`, or any file matching `*swiftui*`, `*swift-testing*`, `*swift-concurrency*`
   - Detect which Apple platform layers the PR touches by looking at file paths:
     - `.swift` files under `**/SwiftUI*/`, `**/View*/`, `**/Screen*/`, or `**/Component*/` → note SwiftUI involvement
     - `.swift` files under `**/Tests*/` or `**/Spec*/` → note testing involvement
     - `.swift` files under `**/Networking*/`, `**/Service*/`, or `**/Storage*/` → note async/concurrency likelihood
     - `.xcstrings`, `.xcassets`, `.plist`, `.xcconfig`, `.xib` changes → note by type
   Collect everything into a single list of paths grouped by purpose (CLAUDE.md vs Swift/iOS skills), and pass it to the review agents in step 4. For each of the three Swift/iOS pro skills, record two facts: (a) is it installed (and the absolute path to its `SKILL.md`), and (b) does the PR touch its domain. These two facts together decide whether the matching domain-specialist agent (Agents #6–#8) runs in step 4 — a specialist runs only when its skill is installed **and** the PR touches its domain. The skills are optional: it is normal for none, some, or all of them to be installed, and a missing skill is never an error.

3. Use a Haiku agent to view the pull request, and ask the agent to return a summary of the change.

4. Then, launch the parallel Sonnet agents below to independently code review the change — the 5 core agents always run, plus up to 3 domain-specialist agents (#6–#8) conditional on step 2's findings. Each agent should return a list of issues, each with its `file:line`, a one-line description, and the rule/reason it was flagged (e.g. CLAUDE.md rule, bug, historical context, prior PR comment, skill rule, etc.):

   a. **Agent #1 — Convention compliance:** Audit the changes for adherence to CLAUDE.md and any discovered guidance files. Check each of the following baseline Swift/iOS conventions against changed lines only — treat them as a starting point and defer to the target project's `CLAUDE.md` wherever it states its own rules (any type, wrapper, or layer names below are illustrative, not prescriptive):
      - **Naming:** protocols use `Any` prefix; no `Implementation` suffix; no single-letter variable names; no file header comments (`//  FileName.swift`, copyright, etc.)
      - **Annotation placement:** property wrappers on the same line as the declaration; `@MainActor`, `@Suite`, `@Test` on the previous line, never inline with the declaration
      - **SwiftUI deprecated APIs:** `foregroundColor` (use `foregroundStyle`), `cornerRadius` (use `clipShape(.rect(cornerRadius:))`), `tabItem` (use `Tab` inside `TabView`), `NavigationView` (use `NavigationStack`), `fontWeight(.bold)` (use `bold()`), `GeometryReader` when `containerRelativeFrame()` or `visualEffect()` would work, `ScrollViewReader` (use `ScrollPosition`/`defaultScrollAnchor`), `UIGraphicsImageRenderer` (use `ImageRenderer`), `showsIndicators:` on scroll views (use `.scrollIndicators(.hidden)`), `onChange` 1-parameter variant (use 2-parameter or zero-parameter closure variant — note: in Swift anonymous closure syntax, `$0` is the first param and `$1` is the second, so a closure using `$1` has two parameters and is already the modern form; only closures using `$0` alone or a single named param are the deprecated 1-parameter variant), `onTapGesture` when `Button` would work, images as button labels without visible text
      - **SwiftUI layout:** no hard-coded padding or stack spacing values without explicit design requirement; no `AnyView` unless absolutely necessary; no `UIScreen.main.bounds`; no computed property views (extract to `View` structs); no `ForEach(Array(items.enumerated()), ...)` — use `ForEach(items.enumerated(), id: \.element.id)`
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
      Note: Apply rules to changed lines only — do not flag pre-existing violations. When the PR touches SwiftUI, tests, or concurrency, leave the deep analysis of those domains to the specialist agents (#6–#8) and focus here on CLAUDE.md and project-convention conformance.

   b. **Agent #2 — Bug scan:** Read the file changes and do a focused scan for obvious Swift/iOS bugs. Avoid reading extra context beyond the changes. Focus on large, impactful bugs, not nitpicks or style issues. Examples to look for:
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

   d. **Agent #4 — Prior PR comments:** Read previous pull requests that touched these files using `gh pr list` and `gh pr view`, and check for any comments on those PRs that may also apply to the current pull request. Flag anything that was previously called out as a problem and appears again.

   e. **Agent #5 — Code comment compliance:** Read the code comments and inline documentation in the modified files. Make sure the changes comply with any guidance, constraints, or intent expressed in those comments. Also flag any newly added comments that describe *what* code does rather than *why* (CLAUDE.md says: no explanatory comments about what the code does).

   The next three are **optional domain specialists**. Each one runs the corresponding installed skill's *own* review process against the changed files — so you get the full depth of `/swiftui-pro`, `/swift-testing-pro`, and `/swift-concurrency-pro` inside this single `/review` run, without invoking them separately. **Spawn a specialist only when step 2 reported that its skill is installed AND the PR touches its domain.** If either is false, skip that agent entirely (do not spawn it, do not mention it). These skills are not required: if none are installed, `/review` runs normally with just the 5 core agents above. Never treat a missing skill as an error or print a warning about it — silently omit the corresponding specialist and continue. Each specialist that does run carries the skill's own priority label — `(high)`, `(medium)`, or `(low)` from its prioritized summary — through with every finding, so the sweep can map it straight to severity.

   f. **Agent #6 — SwiftUI deep review** *(only if `swiftui-pro` is installed and the PR touches SwiftUI):* Read the skill's `SKILL.md` (path from step 2) and follow its review process step by step, loading the `references/` files it points to as needed (api, views, data, navigation, design, accessibility, performance, swift, hygiene). Apply it to the changed SwiftUI files only, on changed lines. Return issues in the same `file:line` + one-line description + rule/reason format, citing the specific skill reference (e.g. `swiftui-pro/references/performance.md`) as the reason.

   g. **Agent #7 — Swift Testing deep review** *(only if `swift-testing-pro` is installed and the PR adds or changes tests):* Read the skill's `SKILL.md` and follow its review process, loading its `references/` files as needed (core-rules, writing-better-tests, async-tests, new-features; migrating-from-xctest only if the PR converts XCTest). Apply it to the changed test files only. Same output format, citing the specific skill reference as the reason.

   h. **Agent #8 — Swift concurrency deep review** *(only if `swift-concurrency-pro` is installed and the PR touches async/concurrency code):* Read the skill's `SKILL.md` and follow its review process, loading its `references/` files as needed (hotspots, actors, structured, unstructured, cancellation, async-streams, bridging, bug-patterns, diagnostics). Apply it to the changed concurrency-relevant files only. Same output format, citing the specific skill reference as the reason.

5. **False-positive sweep (single Sonnet agent).** Combine the issue lists from all step-4 agents into one list and hand it to a single Sonnet agent for an aggressive filter. The agent gets the PR diff, the guidance files from step 2, and the full combined issue list. For each issue, the agent must:
   - Verify the issue is on a line the PR actually modified (not pre-existing).
   - Verify any cited CLAUDE.md or guidance-file rule actually says what the review agent claimed — open the file, find the line, and quote it. For domain-specialist findings (Agents #6–#8), the cited rule lives in the skill's reference files (e.g. `swiftui-pro/references/performance.md`); verify those the same way.
   - Drop nitpicks a senior iOS engineer wouldn't call out in a real PR review.
   - Drop anything a Swift compiler, SwiftLint, or CI would catch on its own.
   - Drop duplicates and merge near-duplicates raised by different step-4 agents into one issue.

   Then for each surviving issue, return its `file:line`, one-line description, the rule/reason, and:
   - `confidence` (0–100): how sure is the agent this is real?
   - `severity`: `high`, `medium`, or `low`
   - `type`: `Bug` | `Style` | `Testing` | `Logic`

   Confidence rubric (give this verbatim to the agent):
   - **0:** False positive, or pre-existing.
   - **25:** Might be real, couldn't verify. Stylistic issues not explicitly in CLAUDE.md land here.
   - **50:** Verified real, but might be a nitpick or rare in practice.
   - **75:** Double-checked, likely hit in practice, important, or directly named in CLAUDE.md.
   - **100:** Confirmed real, will happen frequently, evidence directly confirms.

   Severity rubric (give this verbatim to the agent):
   - **`high`:** breaks functionality, causes incorrect behavior, security issue, data loss, OR a domain-specialist finding the skill ranked `(high)`.
   - **`medium`:** violates a CLAUDE.md or project-convention rule, a meaningful maintainability/refactor issue, OR a domain-specialist finding the skill ranked `(medium)`.
   - **`low`:** minor style, readability, small cleanups, missing tests on non-critical paths, OR a domain-specialist finding the skill ranked `(low)`.
   For domain-specialist findings, the skill's own priority is authoritative — map its `(high)`/`(medium)`/`(low)` label straight to the matching severity.

6. Drop any issue with `confidence < 80`.

7. Output the review as a chat message directly in the conversation (do NOT post to GitHub). Keep output brief. Use the following emojis to prefix each issue line for quick visual scanning: 🔴 for high severity, 🟡 for medium severity, ⚪ for low severity, 🐛 for Bug type, 🎨 for Style type, 🧪 for Testing type, 🔀 for Logic type — combine both the severity and type emoji on each issue (e.g. `🔴 🐛`). Tag each issue with its `[severity · type]` label as well, and link/cite relevant code so the user can navigate to it.

---

## False positives — examples for steps 4 and 5

- Pre-existing issues not touched by the PR
- Something that looks like a bug but is actually intentional or safe in context
- Pedantic nitpicks a senior iOS engineer wouldn't call out
- Issues a compiler, Swift linter (SwiftLint), or type checker would catch — do not run these yourself; CI handles them
- General code quality issues (test coverage, general security) unless explicitly required in CLAUDE.md
- Issues silenced with a `// swiftlint:disable` comment or similar
- Functionality changes that are clearly intentional and directly related to the PR's stated goal
- Real issues on lines the author did not modify

---

## Notes

- Do not attempt to build or compile the app. Build and lint run separately in CI.
- Use `gh` read-only to fetch PR metadata and diffs. Do NOT use `gh pr comment` or any write commands.
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

---

If no issues were found:

---

### Code review

No issues found. Checked for bugs and CLAUDE.md compliance.

---

For code links, prefer a local-style reference: `path/to/File.swift:Lstart-Lend` (clickable in terminals/editors). Alternatively provide a full GitHub URL with the commit SHA: `https://github.com/owner/repo/blob/<full-sha>/path/to/File.swift#Lstart-Lend`. Include at least 1 line of context before and after the flagged line.
