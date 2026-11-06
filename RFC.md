# Draft RFC: Checker-Name Arguments for `[[clang::suppress]]` in the Static Analyzer

We propose changing the Clang Static Analyzer (CSA) to honor the string arguments of `[[clang::suppress("...")]]`, interpreting each argument as a checker name or checker package.

```cpp
[[clang::suppress("core.NullDereference")]]
return p; // suppress null-dereference report

[[clang::suppress("core")]] { // suppress every checker in package "core"
  ...
}

[[clang::suppress("core.DivideZero", "unix.Malloc")]] // suppress any listed name
```

## Context

Today `[[clang::suppress]]` is all-or-nothing in CSA. A bare `[[clang::suppress]]` hides every report in its range. If the attribute has any argument, currently it just ignores the attribute because it's not implemented. This has three consequences:

1. **Over-suppression.** To silence one false positive, users hide all current and future findings in that range, including real bugs from other checkers and from checkers enabled later.
2. **Arguments disarm the suppression.** The attribute already accepts string arguments (`VariadicStringArgument<"DiagnosticIdentifiers">`), but CSA treats any argument as "do not suppress". A bare `[[clang::suppress]]` hides every report in its range, but `[[clang::suppress("core.NullDereference")]]` hides nothing, and the report comes back. Narrowing a suppression therefore turns it off, which is surprising, and users have no way to ask for a targeted suppression.
3. **Reviewability.** Auditing a suppression takes two things: a selective suppression that says what is silenced, and a justification that says why that is safe. A bare suppress gives neither. This RFC provides the first: `suppress("core.NullDereference")` records which finding was judged a false positive. The second is out of scope (see [Future Work](#future-work)).

## Scope

This proposal covers the Clang Static Analyzer only. CSA is the only Clang-based tool that currently honors `clang::suppress`. The Clang Static Analysis Working Group has suggested that clang-tidy and Clang's compiler warnings should support it too, and we discuss that in [Future Work](#future-work). The design allows for this because the attribute takes string names for what to suppress, and those strings could be interpreted by other tools. This proposal does not define a naming or matching scheme shared by all tools.

For context, compiler warnings are currently suppressed with `#pragma clang diagnostic push` / `ignored "-Wfoo"` / `pop`, and clang-tidy uses `NOLINT`, `NOLINTNEXTLINE` and `NOLINTBEGIN`/`NOLINTEND` comments. Neither reads `clang::suppress`. Clang warnings that clang-tidy forwards go through the same `NOLINT` path.

## Design
### Why checker names?

Clang's builtin diagnostics each have a stable identity. Every diagnostic is declared centrally in the `Diagnostic*.td`files, gets a diagnostic ID, and belongs to named groups (`-Wunused-variable`, `-Wformat`).
The Static Analyzer has no equivalent for individual diagnostics. Each checker constructs its own `BugType` (the checker name plus a free-form description and category string) and emits free-form message strings at the report site. There is no stable per-diagnostic ID and there are no diagnostic groups. The description and message text are user-facing prose that often embeds program-specific details and changes freely, so they are not suitable identifiers. The checker is the finest-grained unit that CSA declares and names stably. Users enable and disable analysis with `-analyzer-checker=` and `-analyzer-disable-checker=`, using checker names and packages. 
Choosing checker names therefore makes suppression granularity match enablement granularity. A user who can turn a checker on by name can silence it by the same name.
The limitation is that one checker may emit several different kinds of report, and this design cannot silence one kind without silencing the others. Finer granularity would need a stable per-diagnostic identifier in CSA, which is a much larger change to how checkers declare their reports. That is out of scope for this proposal.
### Semantics

A suppression argument is either a full checker name (`core.NullDereference`) or a checker package (`core`). A package argument matches every checker in that package. Matching works on whole dot-separated components, the same way `-analyzer-checker=` does. Argument strings are always read as CSA checker names or packages.

| Suppress Argument | Reporting Checker | Match |
|---|---|---|
| `core` | `core.NullDereference` | Yes |
| `core.NullDereference` | `core.NullDereference` | Yes |
| `core.Null` | `core.NullDereference` | No |
| `unix.Malloc` | `unix.MallocSizeof` | No |
| `core.NullDereference.X` | `core.NullDereference` | No |

### Checker Renames and Name Stability

Checker names become part of users' source code once they appear in suppressions, so we need to know what happens to those suppressions when a checker is renamed. clang-tidy already faces the same problem with `// NOLINT(check-name)`, so this section looks at how it behaves.

#### What clang-tidy does today

clang-tidy has no rename-specific handling. The text inside `NOLINT(...)` is parsed into a `CachedGlobList`, and a diagnostic is suppressed if `ChecksGlob->contains(DiagName)`, where `DiagName` is the single name the diagnostic was emitted under. Matching is a plain glob (`*` is the only wildcard) against that string. There is no lookup of "previous names".

So in clang-tidy, a `NOLINT` written with a check's old name stops working once the user upgrades to a clang-tidy version where the check has been renamed, and the diagnostic reappears. Updating the `NOLINT` to the new name fixes it for the new version, but then it no longer matches in older versions. clang-tidy does not guarantee any behavior here, and it has no tooling to find `NOLINT`s that refer to names that no longer exist.

#### This proposal

CSA checker names have the same exposure. A rename, or a move between packages such as graduating out of `alpha`, would break suppressions written with the old name. Package arguments (`suppress("pkgA")`) would also stop matching a checker once it moves to `pkgB`. Clang diagnostics avoid this by keeping a list of previous IDs, so suppressions written against an old ID still match the new one. CSA could adopt a similar list of previous checker names. Handling renames and package moves is out of scope for this proposal and is left as future work.

### Backward Compatibility

* Bare `[[clang::suppress]]`: unchanged.
* Existing code with `[[clang::suppress("...")]]` had its arguments ignored, so the attribute suppressed nothing, as if the entire suppression line were absent. Now, the suppressions would match reports corresponding to the checker name argument. This can only decrease the warnings users see, which is what the users expect.

## Alternatives Considered

* **`-analyzer-config` suppression files.** This would add yet another way to configure suppressions and would not fix the actual problem: users already write `[[clang::suppress("name")]]` and expect it to work, and today it suppresses nothing. A config file would also be disconnected from the code and break under refactoring.
* **Raw string-prefix matching.** Rejected: `unix.Malloc` would match `unix.MallocSizeof`, which would surprise users.
* **Teach CSA to honor `NOLINT` instead.** clang-tidy suppresses diagnostics with special source comments. `// NOLINT` silences clang-tidy diagnostics on that line. CSA could recognize the same comments. We rejected this as the solution to the problem in this RFC, for the following reason: CSA already supports the bare `[[clang::suppress]]` attribute. Adding checker names to it extends an existing mechanism, while `NOLINT` would add a second, comment-based mechanism to CSA beside it.
## Future Work

This proposal is deliberately limited to checker names in CSA. Several extensions could build on it without changing its syntax.

* **Other tools.** Make `clang::suppress` work in clang-tidy and for builtin Clang diagnostics, as suggested by the Clang Static Analysis Working Group. Each tool would interpret the string arguments with its own names (`bugprone-foo`, `-Wfoo`). A shared naming and matching scheme across tools, including stable checker names, is a separate design question.
* **Suppression hygiene.** Warn when an argument names no known checker (`suppress("typo.Checker")`) or matches no report. Separately, support justifications: the C++ Core Guidelines spell it `[[gsl::suppress("tag", justification: "message")]]`, which Clang doesn't implement, and adding it to `clang::suppress` would need new argument parsing that fits awkwardly with a variadic string list.
* **Checker categories.** The FIXME in `BugSuppression.cpp` also mentions hashtag-style categories. These are also known as checker labels and would name a group of checkers not tied to a specific checker name; they are being discussed in [Labels and limiting labels of checkers](https://discourse.llvm.org/t/feature-labels-and-limiting-labels-of-checkers/91309). A category is less precise than a checker name, and one category can span many checkers, so a suppression could silence warnings the user didn't intend to hide. Any such mechanism would need separate design and justification.
* **`NOLINT` in CSA.** Recognizing clang-tidy's `NOLINT` comments in CSA, independently of this proposal.
