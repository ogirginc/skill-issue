# Vanilla Rails Domain Modeling

Rich domain models without service layer bloat.

## Model Structure

```ruby
class Recording < ApplicationRecord
  # Associations
  belongs_to :bucket
  belongs_to :creator, default: -> { Current.person }
  delegated_type :recordable, types: %w[Message Document]

  # Concerns group related functionality
  include Copyable
  include Incineratable
  include Eventable

  # Public API - what controllers call
  def copy_to(destination)
    Copier.new(self, destination).copy
  end

  def incinerate
    Incineration.new(self).run
  end
end

# Complex operations get their own class
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

## Concerns for Organization

```ruby
# app/models/recording/copyable.rb
module Recording::Copyable
  extend ActiveSupport::Concern

  def copyable?
    recordable.copyable?
  end

  def copy_to(destination)
    Copier.new(self, destination).copy if copyable?
  end
end
```

## RESTful Controllers: New Controller Instead of Custom Action

Never add a custom action for a verb-like feature. Model the verb as a sub-resource with its own controller using only the standard REST actions.

```ruby
# config/routes.rb
resources :recordings, only: %i[index show destroy] do
  scope module: :recordings do
    resources :copies, only: :create
    resource :incineration, only: :create
  end
end
```

```ruby
# app/controllers/recordings/copies_controller.rb
class Recordings::CopiesController < ApplicationController
  def create
    @recording = Current.account.recordings.find(params[:recording_id])
    @destination = Current.account.buckets.find(params[:bucket_id])

    @copy = @recording.copy_to(@destination)

    redirect_to @copy
  end
end

# app/controllers/recordings/incinerations_controller.rb
class Recordings::IncinerationsController < ApplicationController
  def create
    @recording = Current.account.recordings.find(params[:recording_id])
    @recording.incinerate

    redirect_to @recording.bucket
  end
end
```

Benefits: many small controllers that each stay trivially simple, uniform routing, and each feature gets a natural home for its own before_actions and formats.

## State as Records, Not Booleans

When state needs who/when (and most state eventually does), model it as a record instead of a boolean column. Creating and destroying the record is the state transition.

```ruby
class Post < ApplicationRecord
  has_one :publication, dependent: :destroy

  def published? = publication.present?

  def publish
    create_publication!(creator: Current.person)
  end

  def retract
    publication&.destroy
  end
end

class Publication < ApplicationRecord
  belongs_to :post
  belongs_to :creator, class_name: "Person", default: -> { Current.person }
end
```

You get the timestamp and actor for free (`created_at`, `creator`), a place to hang extra data later (scheduled date, audience), and callbacks/Eventable hooks on the transition itself. Pairs with a RESTful controller: `Posts::PublicationsController#create` / `#destroy`.

## Background Jobs: Thin Wrappers Around Model Methods

Logic lives in the domain model; the job only defers the call. Offer a `_later` variant on the model so callers never touch the job class directly.

```ruby
class Recording < ApplicationRecord
  def incinerate
    Incineration.new(self).run
  end

  def incinerate_later
    Recording::IncinerationJob.perform_later(self)
  end
end

# app/jobs/recording/incineration_job.rb
class Recording::IncinerationJob < ApplicationJob
  queue_as :backend

  def perform(recording)
    recording.incinerate
  end
end
```

Jobs don't inherit request state — set `Current` explicitly inside `perform` if the operation needs it (see `current-attributes.md`).

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
```
