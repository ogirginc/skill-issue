# Delegated Types

Use delegated types when modeling diverse content that shares common operations (versioning, copying, commenting, permissions).

## Structure

```ruby
# Primary table - lightweight metadata
class Recording < ApplicationRecord
  # Generic operations work across all types (copy_to, copyable? — see domain-modeling.md)
  include Copyable

  delegated_type :recordable, types: %w[ Message Document Comment Upload ]

  belongs_to :bucket
  belongs_to :creator, class_name: "Person"
end

# Recordable tables - type-specific content
class Message < ApplicationRecord
  include Recordable

  has_rich_text :content
end

# Shared behavior via concern
module Recordable
  extend ActiveSupport::Concern

  included do
    has_one :recording, as: :recordable, touch: true
  end

  # Capabilities system
  def commentable? = true
  def copyable? = true
  def subscribable? = false
end
```

## Key Principles

- **Recordables are immutable** - Create new versions, don't modify in place
- **Recording holds relationships** - Recordables stay "dumb" with just their data
- **Capabilities system** - Ask recordable `commentable?` rather than hardcoding
- **Efficient copying** - New recording points to same recordable (no duplication)
- **Generic controllers** - Sub-resource controllers (`Recordings::CopiesController`, `Recordings::IncinerationsController`) work for every recordable type

## When to Use

- Content management systems with multiple content types
- Features that apply across types (comments, permissions, versioning)
- Need unified API/querying across diverse content
- Anticipate adding new types over time
