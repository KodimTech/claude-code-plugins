# Database — PostgreSQL, migrations, tenant scoping

Key gems: `acts_as_tenant`, `discard`, `strong_migrations`, `annotaterb`, `ransack`.

---

## Canonical model pattern

**Real example**: `Katalog::Category`.

```ruby
class Katalog::Category < ApplicationRecord
  acts_as_tenant(:core_customer, class_name: "Core::Customer", foreign_key: :core_customer_id)

  include Discardable
  include Ransackable

  has_many :item_categories, class_name: "Katalog::ItemCategory", dependent: :destroy
  has_many :items, class_name: "Katalog::Item", through: :item_categories

  validates :name, presence: true, uniqueness: { scope: :core_customer_id }
end
```

### Observations
- **`acts_as_tenant` first**: before any association or include. Makes it clear from the top that the model is tenant-scoped.
- **Explicit `class_name`**: necessary because `core_customer` doesn't automatically infer to `Core::Customer` (namespace).
- **`foreign_key: :core_customer_id`**: repo convention — **always** this column name for the tenant FK.
- **`include Discardable` and `include Ransackable`**: standard cross-cutting concerns. Use by default.
- **Uniqueness with `scope: :core_customer_id`**: critical. Without the scope, two tenants couldn't have categories with the same name.

---

## `acts_as_tenant` — automatic multi-tenancy

### How it works

- Adds a `default_scope` to the model: `where(core_customer_id: ActsAsTenant.current_tenant.id)`.
- Forces `core_customer_id` on create: automatically taken from the current tenant.
- Cross-tenant queries using `ActsAsTenant.with_tenant(other)` or `ActsAsTenant.without_tenant`.

### What **to** do

```ruby
# Declare in the model
acts_as_tenant(:core_customer, class_name: "Core::Customer", foreign_key: :core_customer_id)

# Use normally — scoping is automatic
Katalog::Category.all    # only current tenant's categories

# In multi-tenant tests
ActsAsTenant.with_tenant(other_customer) do
  Katalog::Category.create!(name: "Other tenant category")
end
```

### What **NOT** to do

```ruby
# ❌ Duplicated manual scope — acts_as_tenant already does it
scope :for_current_customer, -> { where(core_customer_id: ActsAsTenant.current_tenant.id) }

# ❌ Permitting core_customer_id in strong_parameters
params.require(:katalog_category).permit(:name, :core_customer_id)  # NO

# ❌ Manually setting core_customer_id in the controller
params[:core_customer_id] = current_tenant.id  # NO, acts_as_tenant already does it
```

### Exempt models

Only `Core::Customer` (the tenant itself) and `Core::User` (back-office admin, not tenant-scoped) skip `acts_as_tenant`. Everything else uses it.

---

## `discard` — soft delete via `Discardable` concern

### The concern

```ruby
# app/models/concerns/discardable.rb
module Discardable
  extend ActiveSupport::Concern

  included do
    include Discard::Model

    scope :active, -> { kept }
  end
end
```

### Usage

```ruby
class Katalog::Category < ApplicationRecord
  include Discardable
end

category.discard!             # soft delete (sets discarded_at)
category.discarded?           # true
category.kept?                # false

Katalog::Category.kept        # alias of discard's internal scope
Katalog::Category.active      # repo alias (= kept) — use this by default
Katalog::Category.discarded   # soft-deleted records only
```

### Hard rules
- **Never `destroy!`** if the model uses `Discardable`. Use `discard!`.
- **Public query uses `.active`** (not `.all`) to avoid exposing discarded records.
- The model's migration must include `t.datetime :discarded_at` + index:
  ```ruby
  add_index :katalog_categories, :discarded_at
  ```

---

## `Ransackable` — dynamic search

### The concern

Exposes **all** of the model's columns and associations to Ransack:

```ruby
def self.ransackable_attributes(auth_object = nil)
  column_names.map(&:to_s)
end

def self.ransackable_associations(auth_object = nil)
  reflect_on_all_associations.map(&:name).map(&:to_s)
end
```

### Usage

```ruby
class Katalog::Category < ApplicationRecord
  include Ransackable
end

# In the controller
Katalog::Category.active.ransack(params[:q]).result
```

### Rule
- **Always `include Ransackable`** if the model is exposed in any `index`. Don't manually restrict attributes (unless there's a security reason — rare).

---

## Migrations

### Conventions

- **Naming**: `YYYYMMDDHHMMSS_<verb>_<table>.rb`. Verbs: `create`, `add_column`, `add_index`, `add_reference`, `rename`, `drop`, `remove_column`, etc.
- **Version**: use the latest available (`ActiveRecord::Migration[8.1]`).
- **`change`** instead of `up`/`down` when the operation is reversible.

### Example — create a tenant-scoped table

```ruby
class CreateKatalogCategories < ActiveRecord::Migration[8.1]
  def change
    create_table :katalog_categories do |t|
      t.references :core_customer, null: false, foreign_key: { to_table: :core_customers }
      t.string :name, null: false
      t.datetime :discarded_at

      t.timestamps
    end

    add_index :katalog_categories, :discarded_at
    add_index :katalog_categories, [:core_customer_id, :name], unique: true
  end
end
```

### Migration rules

1. **Tenant FK always first**: `t.references :core_customer, null: false, foreign_key: { to_table: :core_customers }`.
2. **Discarded index always** if the model uses `Discardable`: `add_index :table, :discarded_at`.
3. **Tenant-scoped unique index**: `add_index :table, [:core_customer_id, :name], unique: true`. Never global uniqueness without `core_customer_id`.
4. **Simple index** for every FK used in queries (`references` adds one by default — verify).
5. **`null: false`** by default on required columns. Avoid implicit `null: true`.
6. **Explicit defaults** where applicable: `t.string :status, default: 'pending', null: false`.
7. **Polymorphic**: `t.references :integratable, polymorphic: true, null: false` creates `integratable_id` + `integratable_type` + compound index.

### Example — simple add column

```ruby
class AddCallbackTokenToCustomerUsers < ActiveRecord::Migration[8.1]
  def change
    add_column :customer_users, :callback_token, :string
  end
end
```

---

## `strong_migrations` — what it blocks

`strong_migrations` aborts dangerous migrations in production. **Never bypass with `safety_assured`** without explicit justification in a comment.

### Typically blocked operations + solution

| Dangerous operation | Safe solution |
|---|---|
| `add_column :table, :col, :string, default: "x"` on a large table (rewrite) | Split in 2 migrations: `add_column` without default → `change_column_default` → backfill in batches |
| `change_column :table, :col, :new_type` | `add_column :col_new` → backfill → swap → drop old |
| `remove_column` without ignoring first | In PR #1: `self.ignored_columns = %w[col]` in the model. In PR #2: `remove_column` |
| `rename_column` | Same: add new, backfill, ignore old, drop in another PR |
| `add_index` without `algorithm: :concurrently` on a large table | `add_index :table, :col, algorithm: :concurrently` + `disable_ddl_transaction!` |
| `add_foreign_key` without `validate: false` | 2 migrations: `add_foreign_key ... validate: false` → `validate_foreign_key` |
| `add_check_constraint` without `validate: false` | Similar to FK |

### When `safety_assured` is acceptable

- Empty table or one with very few records (e.g., new feature without data yet). Document in a comment: `# safety_assured because: table is empty in all environments`.
- Operation `strong_migrations` flags but the team already validated manually. Document the analysis.

**Never**:
- Silent bypass ("just to make it pass").
- `safety_assured` without an explanatory comment above the block.

---

## `annotaterb` — annotate schemas after migrating

After **every** migration:

```bash
doppler run -- bundle exec annotaterb models
```

This adds schema comments at the top of models and factories:

```ruby
# == Schema Information
#
# Table name: katalog_categories
#
#  id               :bigint           not null, primary key
#  discarded_at     :datetime
#  name             :string           not null
#  ...
```

**Rule**: commit the annotated model together with the migration. Never push a migration without re-annotating.

---

## Factories — FactoryBot

### Conventions

Location: `spec/factories/<module>/<resources>.rb`. **Naming in plural** (`categories.rb`, `bookings.rb`).

### Simple example

```ruby
# spec/factories/katalog/categories.rb
FactoryBot.define do
  factory :katalog_category, class: 'Katalog::Category' do
    association :core_customer
    sequence(:name) { |n| "Category #{n}" }
  end
end
```

### Example with cross-tenant associations and traits

```ruby
# spec/factories/kalendar/bookings.rb
FactoryBot.define do
  factory :kalendar_booking, class: 'Kalendar::Booking' do
    core_customer { association(:core_customer) }
    created_by    { association :customer_user, core_customer: core_customer }
    booked_by     { association :customer_user, core_customer: core_customer }
    kalendar_resource { association :kalendar_resource, core_customer: core_customer }

    start_at { Time.now }
    end_at   { Time.now + 1.hour }
    status   { :pending }

    trait :discarded do
      discarded_at { Time.current }
    end

    trait :confirmed   { status { :confirmed } }
    trait :paid        { status { :paid } }
    trait :cancelled   { status { :cancelled } }
    trait :no_show     { status { :no_show } }
  end
end
```

### Rules
- **Explicit `class: 'Module::Resource'`** when the factory name differs from the class name (snake_case vs Namespace).
- **Tenant-consistent associations**: if a factory creates N related models, they should all share the same `core_customer`. Use explicit `core_customer: core_customer`.
- **`sequence(:field)`** for uniqueness: `sequence(:name) { |n| "Category #{n}" }`.
- **Traits per state**: one trait per enum value or common variant (`:discarded`, `:confirmed`, etc.).
- **Schema annotations at the top**: `annotaterb` also annotates factories.

---

## Seeds

Location: `db/seeds.rb` + auxiliary files in `db/seeds/<module>.rb`.

**Rule**: seeds are for bootstrap data (immutable catalogs like `Core::Subscription`). **Not** for development test data — for that, use factories in the console.

---

## Commands

```bash
# Run migration
doppler run -- bin/rails db:migrate

# Rollback
doppler run -- bin/rails db:rollback

# Full reset (careful in prod, safe in dev)
doppler run -- bin/rails db:reset

# Annotate models after migrating
doppler run -- bundle exec annotaterb models

# Prepare test DB after migration
doppler run --config test -- bin/rails db:test:prepare
```

---

## Checklist for a new migration

- [ ] Naming: `YYYYMMDDHHMMSS_<verb>_<table>.rb`.
- [ ] Correct Rails version in the header (`ActiveRecord::Migration[8.1]`).
- [ ] Tenant-scoped table: has `t.references :core_customer, null: false, foreign_key: { to_table: :core_customers }`.
- [ ] If using `Discardable`: `t.datetime :discarded_at` + `add_index :table, :discarded_at`.
- [ ] Scoped unique indexes: `[:core_customer_id, :field]` when uniqueness is per tenant.
- [ ] `null: false` on required columns.
- [ ] Explicit defaults where applicable.
- [ ] `strong_migrations` doesn't block — or if it does, used a safe alternative (see table).
- [ ] `safety_assured` only with a justifying comment.
- [ ] `doppler run -- bin/rails db:migrate` runs clean.
- [ ] `doppler run -- bundle exec annotaterb models` run, changes committed.
- [ ] Model added with `acts_as_tenant` + `Discardable` + `Ransackable` if applicable.
- [ ] Factory created with tenant-consistent associations.
- [ ] Model spec with associations, validations, unique indexes, valid factory.
