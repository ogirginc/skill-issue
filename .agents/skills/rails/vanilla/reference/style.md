# Code Style

Distilled from 37signals' own [Fizzy STYLE.md](https://github.com/basecamp/fizzy/blob/main/STYLE.md). When unsure how to write something, find similar code in the app and match it. In an existing codebase with its own conventions or RuboCop config, those win.

## Expanded Conditionals Over Guard Clauses

```ruby
# Bad
def todos_for_new_group
  ids = params.require(:todolist)[:todo_ids]
  return [] unless ids
  @bucket.recordings.todos.find(ids.split(","))
end

# Good
def todos_for_new_group
  if ids = params.require(:todolist)[:todo_ids]
    @bucket.recordings.todos.find(ids.split(","))
  else
    []
  end
end
```

Guard clauses get hard to read, especially nested. The exception is an early return right at the start of a method whose main body is several lines long:

```ruby
def after_recorded_as_commit(recording)
  return if recording.parent.was_created?

  if recording.was_created?
    broadcast_new_column(recording)
  else
    broadcast_column_change(recording)
  end
end
```

## Method Order

1. Class methods
2. Public methods, with `initialize` first
3. Private methods

Within that, order methods vertically by invocation: each method comes right after the one that calls it, so the class reads top to bottom.

```ruby
class SomeClass
  def some_method
    method_1
    method_2
  end

  private
    def method_1
      method_1_1
    end

    def method_1_1
      # ...
    end

    def method_2
      # ...
    end
end
```

## Visibility Modifiers

No blank line after `private`; indent the methods under it (as above). A module with only private methods is the exception: `private` at the top, then a blank line, no indent.

```ruby
module SomeModule
  private

  def some_private_method
    # ...
  end
end
```

## Bang Methods

Use `!` only when a counterpart without `!` exists (`save` / `save!`). Don't use it to flag destructive methods — plenty of destructive Ruby and Rails methods have no `!`.

## Controllers and Models

Thin controllers call a rich domain model directly, with nothing in between. Plain Active Record is fine:

```ruby
class Cards::CommentsController < ApplicationController
  def create
    @comment = @card.comments.create!(comment_params)
  end

  private
    def comment_params
      params.expect(comment: [ :body ])
    end
end
```

For anything more involved, give the model an intention-revealing method the controller calls: `@card.gild`. `params.expect` (Rails 8+) responds `400` when params have the wrong shape.

Service and form objects are fine when justified, but don't treat them as a special layer:

```ruby
Signup.new(email_address: email_address).create_identity
```

Endpoints are CRUD on resources — see `domain-modeling.md` for turning verbs into resources.

## Jobs: `_later` and `_now`

`_later` marks a method that enqueues a job. When the job calls back into the same class, the synchronous method gets `_now`. Job classes stay shallow:

```ruby
module Event::Relaying
  extend ActiveSupport::Concern

  included do
    after_create_commit :relay_later
  end

  def relay_later
    Event::RelayJob.perform_later(self)
  end

  def relay_now
    # ...
  end
end

class Event::RelayJob < ApplicationJob
  def perform(event)
    event.relay_now
  end
end
```

## Formatting

Fizzy uses `rubocop-rails-omakase` (Rails' default RuboCop config): double-quoted strings and spaces inside array brackets — `[ :body, :created_at ]`, `%i[ show edit ]`.
