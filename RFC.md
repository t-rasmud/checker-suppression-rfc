# Draft RFC: Checker-Name Arguments for `[[clang::suppress]]` in the Static Analyzer

This proposal makes the Clang Static Analyzer (CSA) honor the string arguments of `[[clang::suppress("...")]]`, interpreting each argument as a checker name or checker package, so that users can silence one checker in a range instead of every checker.

```cpp
[[clang::suppress("core.NullDereference")]]
return p; // suppress null-dereference report

[[clang::suppress("core")]] { // suppress every checker in package "core"
  ...
}

[[clang::suppress("core.DivideZero", "unix.Malloc")]] // suppress any listed name
```

## Context

Today `[[clang::suppress]]` is all-or-nothing in CSA. A bare `[[clang::suppress]]` hides every report in its range. If the attribute has any arguments, currently it just ignores the attribute because it's not implemented. This has three consequences:

1. **Over-suppression.** To silence one false positive, users hide all current and future findings in that range, including real bugs from other checkers and from checkers enabled later.
2. **Arguments disarm the suppression.** The attribute already accepts string arguments (`VariadicStringArgument<"DiagnosticIdentifiers">`), but CSA treats any argument as "do not suppress". A bare `[[clang::suppress]]` hides every report in its range, but `[[clang::suppress("core.NullDereference")]]` hides nothing, and the report comes back. Narrowing a suppression therefore turns it off, which is surprising, and users have no way to ask for a targeted suppression.
3. **Reviewability.** Auditing a suppression takes two things: a selective suppression that says what is silenced, and a justification that says why that is safe. A bare suppress gives neither. This RFC provides the first: `suppress("core.NullDereference")` records which finding was judged a false positive. The second is out of scope. The C++ Core Guidelines spell it as `[[gsl::suppress("tag", justification: "message")]]`, which Clang doesn't implement. Adding it to `clang::suppress` would be a pure addition but needs new argument parsing, and it fits awkwardly with a variadic string list. We leave it as a follow-up that this design doesn't block.

## Scope

* The goal, shared with the Clang Static Analysis Working Group, is for `clang::suppress` to work across the Clang tools: builtin diagnostics, the Static Analyzer and clang-tidy. This RFC specifies the CSA side only.
* This attribute does not currently suppress builtin clang diagnostics. Today the way to suppress those is `#pragma clang diagnostic push` / `ignored "-Wfoo"` / `pop`, which is location-based and understands group names. This is the current state of the implementation, not a restriction of the design. Support for builtin diagnostics could be added later, and this RFC does not prevent it.
* clang-tidy does not currently read this attribute. Its suppression goes through `NOLINT`, `NOLINTNEXTLINE` and `NOLINTBEGIN`/`NOLINTEND`. Clang warnings that clang-tidy forwards go through the same `NOLINT` path.
* This RFC does not define a shared naming or matching scheme for all tools. The unified checker naming scheme and labels are discussed separately (see [Open Questions](#open-questions)), and nothing here should make a shared scheme harder to build later.

## Design

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

So in clang-tidy, `NOLINT` stays valid across a rename only while users stay on the old name. It is not an explicit guarantee, and there is no tooling to find stale `NOLINT`s.

#### Precedent in Clang diagnostics

Clang diagnostics solve this with stable diagnostic IDs plus a list of previous IDs, so suppressions written against a previous ID still match the new one. The same approach would handle checkers moving between packages (for example, graduating out of alpha, or moving from `unix` to `core`), since `pkgA.Bar` becoming `pkgB.Bar` is just a rename. Package-prefix matching (`suppress("pkgA")`) would otherwise stop matching the checker as soon as it moves.

### Backward Compatibility

* Bare `[[clang::suppress]]`: unchanged.
* Existing code with `[[clang::suppress("...")]]` had its arguments ignored, so the attribute suppressed nothing, as if the entire suppression line were absent. Now, the suppressions would match reports corresponding to the checker name argument. This can only decrease the warnings users see, which is what the users expect.

## Alternatives Considered

* **`-analyzer-config` suppression files.** This would add yet another way to configure suppressions and would not fix the actual problem: users already write `[[clang::suppress("name")]]` and expect it to work, and today it suppresses nothing. A config file would also be disconnected from the code and break under refactoring.
* **Raw string-prefix matching.** Rejected: `unix.Malloc` would match `unix.MallocSizeof`, which would surprise users.
* **Teach CSA to honor `NOLINT` instead.** This would not fix the existing expectation that `[[clang::suppress("name")]]` works, since the parameterized attribute would still suppress nothing. Possible and probably worthwhile separately, but this RFC is about CSA's own mechanism, and the attribute already exists and is already honored in its bare form.

## Open Questions

1. **Unknown names.** Should `suppress("typo.Checker")` produce a diagnostic? A CSA-side "unused/unknown suppression" note is possible but out of scope here.
2. **Non-checker identifiers ("hashtags").** The original FIXME also mentions hashtag-style categories (for example `#security`). These are also known as checker labels, and are being discussed in [Labels and limiting labels of checkers](https://discourse.llvm.org/t/feature-labels-and-limiting-labels-of-checkers/91309). This proposal covers checker names only; categories can be layered on later with a distinct syntax that cannot collide with dotted names.
3. **Unified checker naming scheme.** Should the tools converge on one stable checker naming scheme? `clang::suppress` is meant to work across the Clang tools, but CSA checker names (`core.NullDereference`), clang-tidy check names (`bugprone-foo`) and builtin `-W` groups use different schemes. This RFC accepts CSA checker names and packages only. Whether the tools converge on one stable naming scheme, and how renames and moves between packages are handled, is out of scope here.
