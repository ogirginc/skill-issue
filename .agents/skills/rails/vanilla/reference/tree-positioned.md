# Tree and Positioned Concerns

Reusable concerns for parent-child relationships and ordering. Plain Active Record — no gem needed.

## Tree Concern

```ruby
module Tree
  extend ActiveSupport::Concern

  included do
    belongs_to :parent, class_name: name, optional: true
    has_many :children, class_name: name, foreign_key: :parent_id, inverse_of: :parent

    scope :roots, -> { where(parent: nil) }
  end

  def root?
    parent.nil?
  end

  def ancestors
    node, nodes = parent, []
    while node
      nodes << node
      node = node.parent
    end
    nodes
  end
end
```

## Positioned Concern

Requires an integer `position` column, ideally with an index on `[:parent_id, :position]`:

```ruby
add_column :bucket_pins, :position, :integer, null: false
add_index :bucket_pins, [:parent_id, :position]
```

```ruby
module Positioned
  extend ActiveSupport::Concern

  included do
    scope :positioned, -> { order(:position) }

    before_create { self.position ||= (peers.maximum(:position) || 0) + 1 }
  end

  # Works both for reordering within the same parent and moving to a new one.
  def reposition_to(new_position, new_parent: parent)
    transaction do
      # Close the gap left behind in the old scope
      peers.where(position: position..).update_all("position = position - 1")

      self.parent = new_parent

      # Open a gap in the destination scope, then take it
      peers.where(position: new_position..).update_all("position = position + 1")
      update!(position: new_position)
    end
  end

  private

  # Records sharing this record's ordering scope, excluding itself.
  # Scoped by parent_id here (pairs with Tree) — adjust for other scopes.
  def peers
    self.class.where(parent_id: parent_id).where.not(id: id)
  end
end
```

## Usage

```ruby
class Bucket::Pin < ApplicationRecord
  include Tree
  include Positioned

  belongs_to :user
  belongs_to :bucket
end
```
