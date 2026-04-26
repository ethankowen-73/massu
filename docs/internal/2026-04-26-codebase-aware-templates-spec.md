# Codebase-Aware Templates — Engine Spec

**Status**: Authoritative spec for the Plan #2 templating engine.
**Plan**: `/Users/ekoultra/hedge/docs/plans/2026-04-26-massu-codebase-aware-templates.md`
**Module**: `packages/core/src/commands/template-engine.ts`
**Audience**: Implementers + reviewers. Lock all engine semantics here before any extension.

## Goal

Substitute repo-derived values into bundled command templates at install time, **without ever evaluating template content as code**. The engine is string-substitution only.

## Surface (the entire grammar)

| Token form | Meaning |
|------------|---------|
| `{{path.to.var}}` | Look up `path.to.var` in the variables object. Render the value's string form. |
| `{{path.to.var \| default("fallback")}}` | Same as above, but use the `"fallback"` literal if the variable is missing. |
| `\{{` | Literal `{{` — escape sequence, NOT rendered as a token open. |

Whitespace inside `{{ ... }}` is allowed and trimmed. The dot path traverses nested objects via own-property lookups (no prototype walk).

**That is the entire grammar.** No partials, no helpers, no conditionals, no loops, no math, no string functions, no expressions, no inline JS.

## Semantics

1. **String-only output.** The engine reads template text, scans for tokens, and emits substituted strings. Output type is always `string`. No `eval`, no `Function()`, no `vm.*`, no shell, no template-literal interpolation against user content.

2. **Variable lookup uses `Object.hasOwn` only.** Looking up `__proto__.foo` or `constructor.prototype.toString` returns "missing" — the engine MUST NOT traverse the prototype chain. Reaching a non-object on the way to the leaf also returns "missing".

3. **Stringification.** Lookup result is rendered with `String(value)`. `null` → `"null"`, `undefined` → "missing" (triggers default or error). Booleans and numbers render as expected. Arrays/objects render as their default `String(...)` form (callers should not pass non-scalar values; the audit doc lists every variable that is allowed).

4. **Missing without `default`.** Throws `MissingVariableError` (caught by the install caller; install hard-fails for that file only — the rest of the install continues. See Plan §"Error Handling").

5. **`default("...")` accepts ONE double-quoted string literal.** Empty string `""` is allowed. The literal MUST NOT contain unescaped `"` characters. To embed a `"` inside the default, escape with `\"`. No other escape sequences are recognized — the literal is taken byte-for-byte.

6. **No recursive expansion.** A variable VALUE that contains `{{another}}` is rendered as the literal string `{{another}}` — the engine MUST NOT re-scan output for tokens. Untrusted input is data; data is never a template.

7. **No HTML escaping.** Output is markdown destined for `.claude/commands/*.md`, not HTML. A variable value containing `<script>` renders as-is.

8. **Escape sequence.** `\{{` renders as the literal `{{`. The backslash itself is consumed. Any other backslash is preserved verbatim. There is no `\}}` escape — close braces require no escape.

9. **Malformed token.** Any unbalanced `{{` (no closing `}}` on the same line, or a `}}` not preceded by a matching `{{`) throws `TemplateParseError`. **Decision**: `{{{{var}}}}` is treated as `{{` (literal, via `\{{`-style normalization) followed by the token `{{var}}` followed by `}}` (literal close — never consumed without a preceding open). To get a literal `{{var}}`, write `\{{var}}`. Implementers: scan tokens left-to-right; never recurse on output.

10. **Performance.** The engine scans each template in a single linear pass. Templates containing 100,000 instances of `{{var}}` MUST render in <100ms on Node 22 / Apple Silicon. No quadratic behavior, no fixed-point loop.

## Adversarial test cases (TPL-SEC-01..07)

These are mandatory unit tests in `__tests__/template-engine.test.ts`. Each is a single `it(...)` block.

| ID | Input | Expected |
|----|-------|----------|
| TPL-SEC-01 | Body `{{${process.env.PATH}}}` | Literal string `${process.env.PATH}` rendered as data; engine MUST NOT eval. |
| TPL-SEC-02 | Body `{{constructor.constructor("return process")()}}` | Treated as a missing variable lookup (or path-too-long error); engine MUST NOT call `Function()`. |
| TPL-SEC-03 | Variable VALUE `<script>alert(1)</script>` | Rendered verbatim; engine does not auto-escape. |
| TPL-SEC-04 | Variable VALUE `{{another}}` | Rendered as the literal string `{{another}}`; engine MUST NOT re-render. |
| TPL-SEC-05 | Body containing 100,000 `{{var}}` instances | Renders in <100ms. |
| TPL-SEC-06 | Body `{{{{var}}}}` | Renders per rule 9 (literal `{{` + token `{{var}}` + literal `}}`). MUST NOT execute or expand recursively. |
| TPL-SEC-07 | Body `{{__proto__.foo}}` and `{{constructor.prototype.toString}}` | Returns `MissingVariableError` (no `default`) or the `default` value (with `default`); MUST NOT walk prototype. |

Plus a static guard: `grep -nE 'eval\(\|new Function\(\|Function\(\|vm\.\|child_process\|exec\(\|spawn\(' template-engine.ts` returns zero matches.

## Variable scope passed to the engine

When `installAll` renders a template, it passes the union of these objects (later ones win on key collision, but the schema layout below avoids collisions):

```ts
{
  framework: getConfig().framework,   // type, primary, router, orm, ui, languages.*
  paths: getConfig().paths,            // source, aliases.*, web_source, ios_source, ...
  detected: getConfig().detected ?? {} // python.auth_dep, swift.api_client_class, ...
  config: getConfig(),                 // FULL config object (dot-walked); used only by templates that need top-level keys like `toolPrefix`, `python.runtime`, `conventions.claudeDirName`.
}
```

Templates reach into these via dot-walk: `{{paths.source}}`, `{{detected.python.auth_dep | default("get_current_user")}}`, `{{config.toolPrefix}}`.

## Error semantics (caller contract)

| Engine error | Caller behavior |
|--------------|-----------------|
| `MissingVariableError` (no `default`) | The single template install hard-fails. Print the file path, the variable path, and the lookup chain to stderr. Other templates still install. |
| `TemplateParseError` (malformed token, unbalanced braces) | Same — fail this file only, continue with others. |
| Any other thrown error | Bubble up to the install runner and abort the whole install (treat as a bug — file an issue). |

Hash-ordering rule (interaction with the manifest): the rendered output is hashed for the manifest entry, NOT the raw template. Raw template hash and rendered hash diverge for every template that has at least one substituted variable, so rendering MUST happen before `hashContent` on first install AND on safe-upgrade.

## Out of scope

- HTML/Markdown escaping
- Loops or conditionals
- Filters beyond `default`
- Cross-file partials
- Localized output
- Bidirectional template authoring (round-trip via `show-template`)

If any of these are needed, file an issue — do not extend the engine in-band.

## Implementation hints

- The lexer is hand-rolled, not regex-driven (regex has subtle edge cases for `{{{{` and `\{{`).
- Use a state machine: `TEXT`, `IN_TOKEN`, `IN_DEFAULT_STRING`. State transitions are explicit; no recursion.
- Do not call `eval`, `Function`, or `new Function`. Do not import `vm`, `child_process`, `os.execSync`, or `process` for anything other than reading the function input. The only Node modules allowed: NONE (the engine is pure-string).

## References

- Plan: `/Users/ekoultra/hedge/docs/plans/2026-04-26-massu-codebase-aware-templates.md`
- Audit: `docs/internal/2026-04-26-template-variant-audit.md` (Plan #1) and `docs/internal/2026-04-26-codebase-aware-templates-audit.md` (Plan #2 — written in P0-004)
- OWASP ReDoS rules: https://owasp.org/www-community/attacks/Regular_expression_Denial_of_Service_-_ReDoS
