# Code Comments

Distilled from 37signals' [Fizzy](https://github.com/basecamp/fizzy). The general rules from the `code-comments` skill still hold: comment the why, never narrate the change, fix comments your change makes stale. This file covers how that plays out in a Rails app written the 37signals way. In an existing codebase, match its comment density.

## Default to No Comment

37signals code is sparsely commented. About 2 in every 100 lines of Ruby in Fizzy's `app/` are comments, and `Card`, `Card::Closeable` and `CardsController` have none. Names do the explaining: a concern (`Closeable`), a scope (`stalled`), a predicate (`stalled?`), an intention-revealing model method (`card.gild`).

A comment that says what the next lines do is a method waiting to be named:

```ruby
# Bad
def create
  # Mark the card's notification as read
  Current.user.notifications.find_by(card: @card)&.read
end

# Good
def create
  @card.read_by(Current.user)
end
```

## What Earns a Comment

The comments that survive in Fizzy explain something the reader can't see from the Rails code:

- **Not doing the obvious Rails thing.** `# Not using touch so that we can detect attribute change on callbacks` above `update!(last_active_at: Time.current)`.
- **Callback timing.** `# Snapshot IDs in before_destroy since parent record may be deleted by the time after_destroy_commit runs`.
- **Database and concurrency behavior.** `# Processing recipients in order avoids deadlocks if notifications overlap.` above `recipients.sort_by(&:id)`.
- **Cache keys and ETags.** Why the key is what it is, above `fresh_when`: `# Invalidate on the whole set instead of the unread set since the max updated at in the unread set can stay the same when reading old notifications.`
- **Overriding the framework.** `# Need to set in the included block so that it overrides Action Text's`.
- **A rescue that swallows the error.** One line saying why that's safe:

```ruby
def assign(user)
  assignments.create!(assignee: user)
rescue ActiveRecord::RecordNotUnique
  # Already assigned
end
```

- **Logic duplicated across Ruby and JavaScript.** Point each side at the other, so whoever changes one finds the other:

```ruby
# Keep in sync with #isStalled in app/javascript/controllers/bubble_controller.js
def stalled?
```

```javascript
// Keep in sync with Card::Stallable#stalled? in app/models/card/stallable.rb
get #isStalled() {
```

## Header Comments

A short paragraph above a concern, job or class whose purpose or mechanism the name doesn't carry:

```ruby
# Serializes the current account into job data so that jobs run
# within the correct account context via Current.with_account.
#
# Account resolution is deferred to an around_perform callback so
# that missing accounts raise DeserializationError inside the
# execution path where discard_on can handle it.
module AccountTenanted
```

A patch to Rails, in `config/initializers` or wherever the app keeps its Rails extensions, always gets one: what it changes, and why the default doesn't work for this app. A job meant to be run by hand gets a `Usage:` block with the call indented by two spaces.

## Concern Contracts and Hooks

When a concern needs the including model to supply methods, list them at the bottom of the concern:

```ruby
module Searchable
  extend ActiveSupport::Concern

  # ...

  # Models must implement these methods:
  # - search_title: returns title string or nil
  # - search_content: returns content string
  # - searchable?: returns whether this record should be indexed
end
```

Label a default meant to be overridden: `# Template method` above it, or `# Override to ...` on an empty hook. Here a label is enough; it doesn't need a full sentence.

## FIXME and TODO

`FIXME` marks code known to be wrong or in the wrong place; `TODO` marks unfinished work. Say what and why, and delete it when it's done:

```ruby
# FIXME: This is too inefficient to have part of a destroy transaction.
# Need to find a way to use a job or a single query.
def remove_from_filters
```

`# FIXME: Upstream this fix to turbo-rails` above a workaround tells the next reader it's temporary and where it belongs.

## Comments in Views

Use `<%# %>`, never `<!-- -->`, which ships to every browser.

In a template that calls `cache`, comments are part of the cache key. The key includes a digest of the template's source and the partials it renders, so editing anything in the file, comments included, expires its fragments. Two comment formats rely on this.

**`Template Dependency:`** declares a partial the digest can't find on its own, such as one rendered inside a helper. Rails parses it with `/# Template Dependency: (\S+)/`, so type it exactly, one path per line; wildcards work:

```erb
<%# Template Dependency: events/* %>
<%= render_categorizable_events @person.events %>
```

**`Helper Dependency Updated:`** is plain text. The digest doesn't track helpers, so when you change a helper used inside a cached fragment, add or update this line with the helper's name and the date:

```erb
<% cache comment do %>
  <%# Helper Dependency Updated: avatar_image_tag 2025-12-15 %>
  <%= avatar_image_tag comment.creator %>
<% end %>
```

Don't delete these as cleanup. Without the `Template Dependency:` line, changing that partial stops expiring the fragment.

## Stimulus Controllers

Same rules as Ruby. Some longer Fizzy controllers group members under `// Lifecycle`, `// Actions` and `// Private` labels; most don't. Use them only in a controller long enough to need them.

## Leave These Out

None of these appear in Fizzy's app code:

- **Labels for Rails DSL.** `# Associations`, `# Validations`, `# Callbacks`. Group with blank lines instead.
- **Scaffold route comments.** `# GET /cards/1` above an action. Routes and REST already say it.
- **Schema annotations.** `# == Schema Information` blocks. `db/schema.rb` is the record.
- **YARD tags in app code.** `@param` and `@return` restate what names and tests already say.

## Framework Code Is Different

Library code in `lib/` written as if it were part of Rails (Fizzy's `lib/action_pack/passkey`) is documented the way Rails is: RDoc with `== Usage` headings, examples indented two spaces, and code in `+plus signs+`. App code doesn't get that treatment.

Generator comments in `config/` and `ApplicationJob` are Rails' own documentation. Leave them, but don't add commented-out code of your own.
