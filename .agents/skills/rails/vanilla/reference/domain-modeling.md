# Vanilla Rails Domain Modeling

Rich domain models without service layer bloat.

## Model Structure

```ruby
class Recording < ApplicationRecord
  # Concerns group related functionality and carry the public API controllers call
  include Copyable, Eventable, Incineratable

  belongs_to :bucket
  belongs_to :creator, class_name: "Person", default: -> { Current.person }
  delegated_type :recordable, types: %w[ Message Document ]
end
```

Keep the class body thin; behavior lives in concerns under `app/models/recording/`. Don't redefine a concern's method in the class body — the class's version silently wins over the module's.

## Concerns for Organization

```ruby
# app/models/recording/copyable.rb
module Recording::Copyable
  extend ActiveSupport::Concern

  def copyable?
    recordable.copyable?
  end

  def copy_to(destination)
    # Fully qualified: inside `module Recording::Copyable`, a bare `Copier`
    # doesn't search `Recording` and raises NameError.
    Recording::Copier.new(self, destination).copy if copyable?
  end
end

# app/models/recording/copier.rb — complex operations get their own class
class Recording::Copier
  def initialize(recording, destination)
    @recording = recording
    @destination = destination
  end

  def copy
    Event.suppress do
      @destination.record(@recording.recordable, creator: Current.person)
    end
  end
end
```

This is not a service object in disguise: the entry point stays on the model (`recording.copy_to(destination)`), and the operation class lives in the model's namespace as an implementation detail.

Name concerns for the capability they add (`Copyable`, `Incineratable`, `Closeable`) and keep each one self-contained: its associations, scopes, and methods together.

## Preloading: One Named Scope

Name the associations a list needs once, and chain that scope wherever records get listed. Fizzy's card queries end in `.preloaded` the same way:

```ruby
class Recording < ApplicationRecord
  scope :preloaded, -> { includes(:creator, :bucket, :recordable) }
end

@recordings = @bucket.recordings.preloaded
```

## RESTful Controllers: New Controller Instead of Custom Action

Never add a custom action for a verb-like feature. Turn the verb into a noun and give it its own controller using only the standard REST actions.

| Verb | Resource | Requests |
|------|----------|----------|
| copy | `resources :copies` | `POST` |
| incinerate | `resource :incineration` | `POST` |
| publish / retract | `resource :publication` | `POST` / `DELETE` |
| close / reopen | `resource :closure` | `POST` / `DELETE` |
| pin / unpin | `resource :pin` | `POST` / `DELETE` |

Use singular `resource` for state the parent either has or doesn't (a post has one publication); plural `resources` when each request creates a new record (copies).

```ruby
# config/routes.rb
resources :recordings, only: %i[ index show destroy ] do
  scope module: :recordings do
    resources :copies, only: :create
    resource :incineration, only: :create
  end
end
```

Sub-resource controllers share the parent lookup through a scoping concern:

```ruby
# app/controllers/concerns/recording_scoped.rb
module RecordingScoped
  extend ActiveSupport::Concern

  included do
    before_action :set_recording
  end

  private
    def set_recording
      @recording = Current.account.recordings.find(params[:recording_id])
    end

    def ensure_can_administer_recording
      head :forbidden unless Current.person.can_administer?(@recording)
    end
end
```

```ruby
# app/controllers/recordings/copies_controller.rb
class Recordings::CopiesController < ApplicationController
  include RecordingScoped

  before_action :ensure_copyable

  def create
    @copy = @recording.copy_to(Current.account.buckets.find(params[:bucket_id]))
    redirect_to @copy
  end

  private
    def ensure_copyable
      head :forbidden unless @recording.copyable?
    end
end

# app/controllers/recordings/incinerations_controller.rb
class Recordings::IncinerationsController < ApplicationController
  include RecordingScoped

  before_action :ensure_can_administer_recording

  def create
    @recording.incinerate_later
    redirect_to @recording.bucket
  end
end
```

Benefits: many small controllers that each stay trivially simple, uniform routing, and each feature gets a natural home for its own before_actions and formats.

Always look records up through the tenant, or through what the current person can access — never `Recording.find(params[:id])`.

## Authorization: Model Predicates, Controller Checks

No policy objects or authorization gems. The model answers "may this person do this?"; the controller asks in a `before_action` and responds `403`, as `ensure_can_administer_recording` does above:

```ruby
class Person < ApplicationRecord
  def can_administer?(recording)
    admin? || recording.creator == self
  end
end
```

Capability checks work the same way (`ensure_copyable` above). Keep the model's own guard as well — `copy_to` does nothing unless the recording is copyable. The controller check gives the client the right response; the model guard protects every other caller (jobs, console, other models).

## State as Records, Not Booleans

When state needs who/when (and most state eventually does), model it as a record instead of a boolean column. Creating and destroying the record is the state transition.

```ruby
class Post < ApplicationRecord
  include Eventable

  has_one :publication, dependent: :destroy

  scope :published, -> { joins(:publication) }
  scope :draft, -> { where.missing(:publication) }

  def published?
    publication.present?
  end

  def publish
    unless published?
      transaction do
        create_publication!
        track_event :published
      end
    end
  end

  def retract
    if published?
      transaction do
        publication.destroy!
        track_event :retracted
      end
    end
  end
end

class Publication < ApplicationRecord
  belongs_to :post, touch: true
  belongs_to :creator, class_name: "Person", default: -> { Current.person }
end
```

You get the timestamp and actor for free (`created_at`, `creator`), a place to hang extra data later (scheduled date, audience), and callbacks/Eventable hooks on the transition itself. The scopes come straight from `joins` / `where.missing`, the guards make `publish` and `retract` safe to call twice, and the transaction keeps the state and its event together. Pairs with `resource :publication`: `Posts::PublicationsController#create` / `#destroy`.

## Background Jobs: Thin Wrappers Around Model Methods

Logic lives in the domain model; the job only defers the call. `_later` marks the method that enqueues the job, and when the job calls back into the same class, the synchronous method gets `_now` (see `style.md`). Callers use these methods and never touch the job class directly.

```ruby
# app/models/recording/incineratable.rb
module Recording::Incineratable
  extend ActiveSupport::Concern

  def incinerate_later
    Recording::IncinerationJob.perform_later(self)
  end

  def incinerate_now
    Recording::Incineration.new(self).run
  end
end

# app/jobs/recording/incineration_job.rb
class Recording::IncinerationJob < ApplicationJob
  queue_as :backend

  def perform(recording)
    recording.incinerate_now
  end
end
```

When a model change triggers the job, enqueue it from a commit callback (`after_create_commit :relay_later`). Also make jobs enqueued inside a transaction wait for it to commit; otherwise a fast worker can run before the data it needs exists:

```ruby
# config/initializers/active_job.rb
ActiveSupport.on_load(:active_job) do
  self.enqueue_after_transaction_commit = true
end
```

Jobs don't inherit request state. Restore the tenant automatically (see "Carrying the Account into Jobs" in `current-attributes.md`) and pass anything else the job needs explicitly.

## What NOT to Do

```ruby
# AVOID: Anemic models + service layers
class RecordingCopyService
  def call(recording_id, destination_id)
    recording = Recording.find(recording_id)
    destination = Bucket.find(destination_id)
    # All logic here, model is just data
  end
end

# AVOID: Separate "application layer"
class CopyRecordingUseCase
  def execute(params)
    # Unnecessary indirection
  end
end

# AVOID: Custom controller actions for verbs
resources :recordings do
  member do
    post :copy        # -> Recordings::CopiesController#create instead
    post :incinerate  # -> Recordings::IncinerationsController#create instead
  end
end

# AVOID: Unscoped lookups
@recording = Recording.find(params[:recording_id])  # -> Current.account.recordings.find(...)
```
