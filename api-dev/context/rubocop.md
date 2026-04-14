# RuboCop — Rails Omakase

core-web-api uses **Rails Omakase** (`rubocop-rails-omakase`) as the style base, with no custom overrides.

```yaml
# .rubocop.yml
inherit_gem: { rubocop-rails-omakase: rubocop.yml }
```

There are no custom rules — if RuboCop flags something, it's an official Omakase rule.

---

## Mandatory flow before every commit

### 1. Safe autocorrect first

```bash
doppler run --config test -- bundle exec rubocop -A <file_or_dir>
```

`-A` applies ALL autocorrections (including "unsafe"). In Omakase, the "unsafe" ones are usually safe — run, review the diff.

Typical fixes `-A` handles:
- Spaces and quotes
- `Layout/*` (indentation, line length)
- `Style/StringLiterals` (double quotes by default in Omakase)
- `Style/HashSyntax` (symbol key syntax)

### 2. Manual for what remains

If offenses remain after `-A`, they're **logic/design** problems that require human judgment:
- `Metrics/MethodLength` — break down the method.
- `Metrics/AbcSize` — simplify the logic.
- `Lint/*` — usually indicates a real bug (shadow variable, empty rescue, etc.).

**Rule**: don't silence with `disable` — **fix the root cause**.

### 3. Re-verify

```bash
doppler run --config test -- bundle exec rubocop <file_or_dir> --format simple
```

It must come out clean. If not, repeat step 2.

---

## `# rubocop:disable` — when it's acceptable

Only when:
1. **The cop doesn't apply to the context**: e.g., a long method in an auto-generated mapper.
2. **Fixing would make the code worse**: e.g., breaking a method to satisfy `Metrics/MethodLength` loses clarity.

Always with a **comment above explaining why**:

```ruby
# rubocop:disable Metrics/MethodLength
# Long method justified: Stripe webhook handler with 12 distinct event types —
# extracting would create 12 near-empty classes with no reuse.
def handle_stripe_webhook(event)
  case event.type
  when 'charge.succeeded' then ...
  # ...
  end
end
# rubocop:enable Metrics/MethodLength
```

**Never**:
- Disable without a comment.
- Disable an entire file (`# rubocop:disable all`).
- "Preemptive" disable because "maybe in the future".

---

## `.rubocop_todo.yml` — **don't touch** in feature PRs

`.rubocop_todo.yml` contains pre-existing offenses frozen. It's technical debt under control.

**Hard rules**:
- **Never** modify `.rubocop_todo.yml` in a feature/bugfix PR. If a cop flags new code, fix it — don't add it to the todo.
- **Never** regenerate `.rubocop_todo.yml` (e.g., `rubocop --regenerate-todo`) in a non-chore PR. That hides new offenses.
- If you need to pay down debt from the todo, do it in a dedicated PR (`chore/`) with explicit scope.

---

## Reference commands

```bash
# Safe autofix
doppler run --config test -- bundle exec rubocop -A

# Autofix only in changed files
doppler run --config test -- bundle exec rubocop -A $(git diff --name-only --diff-filter=ACMR origin/main | grep '\.rb$')

# Check without fix
doppler run --config test -- bundle exec rubocop --format simple

# Check a specific file
doppler run --config test -- bundle exec rubocop app/recorders/katalog/categories/create_recorder.rb

# List all active cops
bundle exec rubocop --show-cops
```

---

## Warning signals

If any of these occur, stop and review:

1. **Many `# rubocop:disable`** in a PR → you're fighting the repo's style; check if the design has a deeper problem.
2. **`Metrics/AbcSize`** flagging a method → almost certainly extractable into private methods or a Service.
3. **`Rails/SkipsModelValidations`** (e.g., `update_all`, `update_columns`) → use with reason, never out of blind speed.
4. **`Lint/UnusedMethodArgument`** → clean up, don't rename to `_var` except in interface overrides.
5. **`Rails/OutputSafety`** (use of `html_safe` or `raw`) → verify input is sanitized. API-only app: this cop should be rare.

---

## RuboCop checklist before reporting a PR

- [ ] `bundle exec rubocop -A <changed_files>` applied.
- [ ] Re-run without `-A` comes out clean (0 offenses).
- [ ] Any `# rubocop:disable` has a justifying comment.
- [ ] `.rubocop_todo.yml` **not** modified.
- [ ] No `# rubocop:disable all` on entire files.
- [ ] `bin/ci` (which includes rubocop) passes green.
