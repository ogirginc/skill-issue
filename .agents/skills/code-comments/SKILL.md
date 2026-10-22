---
name: code-comments
description: Write code comments that explain why — intent, constraints and surprises — and never restate what the code already says. Use whenever writing or editing code in any language (new features, bug fixes, refactors, scripts), and whenever deciding whether to add, keep or change a comment, even if the user doesn't mention comments.
---

# Code Comments

The code already says *what* it does. A comment earns its place by saying what the code can't: *why* it's written this way.

## Before adding a comment

Ask whether a competent reader of this language would miss something without it. If not, leave it out: a comment that repeats the code is one more thing to read, and one more thing to go stale when the code changes.

## Comment the why

Good reasons to write a comment:

- **A non-obvious choice.** Why this approach and not the obvious one.
- **An outside constraint.** An API quirk, a database or concurrency behavior, a rate limit, a product or legal rule the code can't express.
- **Something that looks wrong but isn't.** A deliberately ignored error, an intentionally unawaited call, a magic number and where it comes from.
- **A workaround.** What it works around, and why it's still needed.
- **An invariant.** Something a future editor must keep true that the code doesn't enforce.

```ruby
# The bank's sandbox allows one request every 2 seconds per API key.
sleep 2
```

```typescript
// Not awaited on purpose: analytics must never delay the response.
trackEvent("checkout", order.id)
```

```python
# The payments API answers declines with HTTP 200 and an error body,
# so the status code alone doesn't mean the charge went through.
if error := response.json().get("error"):
    raise PaymentDeclined(error)
```

## Leave these out

- **Restating the code.** `# find the user by email`, `# loop over the orders`, `# return the result`.
- **Narrating the change.** `# Changed to a set to fix duplicates`, `# Added per review`, `# New implementation`. That history belongs in the commit message; in the code it outlives the context that made it mean anything.
- **Talking to the person you're working with.** `# You may want to adjust this timeout`. The comment's reader is the next maintainer, not the current conversation.
- **Commented-out code.** Delete it; version control keeps it.
- **Placeholders.** `# ... rest of the method unchanged ...` never belongs in a real file.

```ruby
# Bad: restates the code
# Find the user by email
user = User.find_by(email: email)

# Good: the code already says it
user = User.find_by(email: email)
```

## How to write one

- **State the reason as a sentence.** `# Must be idempotent: the provider retries webhooks for up to 3 days.` tells the reader far more than `# idempotent`.
- **Be specific.** Name the constraint ("MySQL won't use the index with this OR") instead of a vague "for performance".
- **Describe the code as it is now**, in the present tense.
- **Keep it close and short.** Put it directly above the code it explains, in a line or two. If the explanation needs a paragraph, it may belong in a commit message or doc, with a one-line pointer here.

## When editing existing code

- **Fix comments your change makes wrong.** A stale comment is worse than none, because readers trust it.
- **Comments inside code you rewrite are part of your change.** Apply these rules to them.
- **Leave other comments alone.** Rewriting comments you weren't asked about makes the diff noisy and hides the real change.
- **Match the codebase.** Follow its comment density, its doc-comment format (YARD, JSDoc, docstrings) and its TODO style. If every public method in the file has a doc comment, a new one gets one too.
