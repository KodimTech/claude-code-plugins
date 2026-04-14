# Testing — RSpec, FactoryBot, shared contexts

The 3 hard testing rules in core-web-api. **Non-negotiable**:

1. **Spec-driven**: RED → GREEN. Write specs first, verify they fail, implement until they pass.
2. **Only model specs touch the DB**. Recorders, services, controllers, jobs, mailers, LLM tools → `instance_double`, `double`, stubs, mocks. Never `create(:factory)` in non-model specs except when justified with an explicit comment.
3. **Diff coverage verified with `undercover --compare origin/main`** — blocking. If it reports uncovered lines, success must not be declared.

Global coverage minimum: **85%**. Triple test mandatory per component: happy path, error path, multi-tenant isolation.

---

## Spec organization

| Type | Location | `type:` | Touches DB | Guide |
|---|---|---|---|---|
| Model | `spec/models/<module>/<resource>_spec.rb` | `:model` | **Yes** | see Models section |
| Recorder | `spec/recorders/<module>/<resources>/<op>_recorder_spec.rb` | — | **No** | `recorders.md` |
| Service | `spec/services/<module>/...` | — | **No** | `services.md` |
| LLM Tool | `spec/services/kathy/llm/tools/...` | — | **No** | `services.md` |
| Controller (request) | `spec/requests/api/v1/<audience>/<module>/<resources>_spec.rb` | `:request` | **No** by default | `controllers.md` |
| Job | `spec/jobs/<module>/<resource>_job_spec.rb` | `:job` | **No** | see Jobs section |
| Mailer | `spec/mailers/<module>/<resource>_mailer_spec.rb` | `:mailer` | **No** | see Mailers section |

---

## Matrix: "can my spec persist?"

| Spec | `create(:factory)` | `build(:factory)` | `instance_double` |
|---|---|---|---|
| Model | ✅ — required by `shoulda-matchers` (`validate_uniqueness_of`, etc.) | ✅ for factory validity | N/A |
| Recorder | ❌ | ❌ | ✅ **required** (for `::Model` and passed resource) |
| Service / LLM Tool | ❌ | ❌ | ✅ **required** |
| Controller (Unit) | ❌ | ❌ | ✅ **required** (for Recorders/Serializers) |
| Controller (Integration) | ⚠️ **justified exception** — comment required | ❌ | ❌ |
| Job | ❌ | ❌ | ✅ **required** |
| Mailer | ❌ | ✅ for the email recipient | ✅ for other deps |

**Rule**: if tempted to `create(:factory)` in a non-model spec, write in the spec the comment `# NOTE: justified because <reason>` and double-check you really can't mock it. Default = you can.

---

## Available shared contexts

Location: `spec/support/context/`. **Use them, don't invent new ones**.

### `core_customer` (base)

Creates the tenant and sets it as current. Mocks welcome mailer and Cloudflare.

```ruby
RSpec.shared_context 'core_customer' do
  let_it_be(:core_customer) do
    customer = nil
    RSpec::Mocks.with_temporary_scope do
      allow(::Admin::CoreCustomerMailer).to receive_message_chain(:welcome, :deliver_now)
      allow_any_instance_of(::Integrations::Cloudflare::Pages::Domains).to receive(:perform)
      customer = create(:core_customer)
    end
    customer
  end

  before_all { ActsAsTenant.current_tenant = core_customer }
  before { ActsAsTenant.current_tenant = core_customer }
end
```

**Usage**:
```ruby
RSpec.describe Katalog::Item, type: :model do
  include_context 'core_customer'
  # core_customer available, tenant already set
end
```

### `core_customer_with_subscriptions`

Extends `core_customer` with preloaded subscriptions (Reservas, Kampus). Use when the spec depends on subs existing.

### `api_core_customer`

Extends `core_customer` + sets `host!` to the subdomain:

```ruby
RSpec.shared_context 'api_core_customer' do
  include_context 'core_customer'

  before { host! "#{core_customer.subdomain}.example.com" }
end
```

Use in request specs that don't require real JWT auth but need tenant resolution via subdomain.

### `api_authentication`

The most complete: `core_customer` + `host!` + `customer_user` + `authorization` header.

```ruby
RSpec.shared_context 'api_authentication' do
  include_context 'api_core_customer'

  let_it_be(:customer_user, reload: true) { create(:customer_user, core_customer: core_customer) }
  let(:authorization) do
    customer_user.generate_jwt
    "Bearer #{customer_user.jwt}"
  end
end
```

**Usage in integration request specs (exception)**:
```ruby
RSpec.describe 'API V1 Customer Katalog Categories', type: :request do
  include_context 'api_authentication'

  it 'returns 200 authenticated' do
    get '/api/v1/customer/katalog/categories', headers: { Authorization: authorization }
    expect(response).to have_http_status(:ok)
  end
end
```

---

## FactoryBot: `build` > `create`

When you DO use FactoryBot (model specs, or justified exceptions in integration), prefer `build` over `create`:

| Method | Persists | Use |
|---|---|---|
| `build(:factory)` | No | Factory validity, validations, in-memory construction |
| `build_stubbed(:factory)` | No (stubs ID) | Complex associations without DB — useful for "pretend it exists" |
| `create(:factory)` | Yes | Only when you need real persistence (associations, uniqueness, etc.) |

**Rule**: always start with `build`. Only escalate to `create` if the test requires it (e.g., shoulda `validate_uniqueness_of` does require it).

---

## test-prof: `let_it_be` and `before_all`

Available via `test_prof/recipes/rspec/let_it_be` + `before_all`.

| Helper | Use | Advantage |
|---|---|---|
| `let_it_be(:x) { ... }` | Data shared across **all** examples in a `describe` | Created once per group, not per example |
| `let_it_be(:x, reload: true) { ... }` | Same, but reloads before each example | Useful if the spec mutates `x` |
| `before_all { ... }` | Setup once per `describe` | Faster than `before` |

**Rule**: in model specs with shared data, use `let_it_be` by default. `let` only if the value is recalculated per test.

---

## Patterns per type

Full details live in the type-specific context files. Here are minimal examples only.

### Models — DB allowed, shoulda-matchers

```ruby
RSpec.describe Katalog::Item, type: :model do
  include_context 'core_customer'

  describe 'associations' do
    it { should belong_to(:core_customer) }
    it { should have_many(:katalog_item_categories).dependent(:destroy) }
  end

  describe 'validations' do
    subject { build(:katalog_item, core_customer: core_customer) }
    it { should validate_presence_of(:name) }
    it { should validate_uniqueness_of(:name).scoped_to(:core_customer_id) }
  end

  describe 'enums' do
    it { should define_enum_for(:status).with_values(available: 'available', unavailable: 'unavailable') }
  end

  describe 'factories' do
    it { expect(build(:katalog_item, core_customer: core_customer)).to be_valid }
  end
end
```

Minimum coverage for a model spec:
- Associations (`belong_to`, `have_many`, `has_one`, nested attributes)
- Validations (presence, uniqueness with `subject`, length, numericality, inclusion)
- Enums
- Unique indexes (`have_db_index(:col).unique(true)`)
- Important callbacks
- Custom scopes
- Public methods with logic
- Valid factory

### Recorders — instance_double, no DB

See `recorders.md` — full pattern. Summary:

```ruby
subject(:recorder) { described_class.new(params: params) }
let(:params) { { name: 'X' } }
let(:model) { instance_double(::Katalog::Category) }

it 'creates and adds event' do
  expect(::Katalog::Category).to receive(:create!).with(params).and_return(model)
  expect { expect(recorder.record).to eq(model) }
    .to change { recorder.events }.from([]).to([:katalog_category_created])
end
```

### Services / LLM Tools — instance_double, stub recorders

See `services.md` — full patterns. Key: **never** `create(:booking)`, use `instance_double(::Kalendar::Booking)` and `instance_double(::Kalendar::Bookings::CreateRecorder, record: booking_double)`.

### Controllers (request specs) — Unit by default

See `controllers.md`. Summary of Unit approach:

```ruby
before do
  host! 'test.example.com'
  allow_any_instance_of(::ApplicationController).to receive(:set_current_user).and_return(true)
  allow_any_instance_of(::ApplicationController).to receive(:set_tenant).and_return(true)
end
```

### Jobs — instance_double + enqueuing

```ruby
RSpec.describe Kommerce::OrderConfirmationJob, type: :job do
  let(:order) { instance_double(::Kommerce::Order, id: 1) }

  describe '#perform' do
    it 'delegates to mailer' do
      expect(::Kommerce::Order).to receive(:find).with(1).and_return(order)
      expect(::Kommerce::OrderMailer).to receive(:confirmation).with(order).and_return(double(deliver_later: true))
      described_class.perform_now(1)
    end
  end

  describe 'enqueuing' do
    it 'enqueues with IDs, not objects' do
      expect { described_class.perform_later(1) }
        .to have_enqueued_job(described_class).with(1)
    end

    it 'uses default queue' do
      expect { described_class.perform_later(1) }
        .to have_enqueued_job.on_queue('default')
    end
  end
end
```

**Rules**:
- Jobs receive **IDs**, not objects. Test with `.with(1)`, not `.with(order)`.
- Test `perform_now` (logic) separate from `perform_later` (enqueuing).
- `have_enqueued_job` is the canonical matcher.

### Mailers — build (not create), no delivery

```ruby
RSpec.describe Kommerce::OrderMailer, type: :mailer do
  let(:user) { build(:customer_user, email: 'test@example.com', first_name: 'John', last_name: 'Doe') }
  let(:mail) { described_class.confirmation(user) }

  describe '#confirmation' do
    it 'renders headers' do
      expect(mail.subject).to eq('Order Confirmation')
      expect(mail.to).to eq(['test@example.com'])
    end

    it 'renders body with user name' do
      expect(mail.body.encoded).to include('John Doe')
    end

    it 'enqueues the email via deliver_later' do
      expect { described_class.confirmation(user).deliver_later }
        .to have_enqueued_mail(described_class, :confirmation).with(user)
    end
  end
end
```

**Rules**:
- `build` not `create` — you don't need to persist the recipient.
- Test headers (subject, to, from) and body.
- If multipart: test `mail.html_part` and `mail.text_part` separately.
- Preview in `spec/mailers/previews/<module>/<resource>_mailer_preview.rb`.

---

## Multi-tenant isolation — **mandatory**

Every component touching a scoped model must have a spec verifying isolation:

```ruby
context 'tenant isolation' do
  let_it_be(:other_customer) { create(:core_customer, subdomain: 'other') }
  let_it_be(:other_booking) do
    ActsAsTenant.with_tenant(other_customer) do
      other_user = create(:customer_user, core_customer: other_customer)
      create(:kalendar_booking, booked_by: other_user, core_customer: other_customer)
    end
  end

  it 'only returns current tenant data' do
    result = tool.execute
    expect(result).not_to include(other_booking.id.to_s)
  end
end
```

**When**: any Query Service, LLM Tool, controller index that filters/queries a model with `acts_as_tenant`.

---

## Useful commands

All tests require `doppler run --config test --` prefix (without it, ENV vars are missing):

```bash
# Whole suite
doppler run --config test -- bundle exec rspec

# One file
doppler run --config test -- bundle exec rspec spec/models/katalog/item_spec.rb

# A specific test
doppler run --config test -- bundle exec rspec spec/models/katalog/item_spec.rb:23

# With coverage
doppler run --config test -- COVERAGE=true bundle exec rspec

# Diff coverage (blocking before reporting success)
doppler run --config test -- bundle exec undercover --compare origin/main

# Full pipeline (rspec + rubocop + brakeman + signoff)
bin/ci
```

**Forgetting the `doppler run --config test --` prefix** → tests fail inconsistently with ENV var errors. Always include it.

---

## Undercover — diff coverage

After rspec passes green, **always** run:

```bash
doppler run --config test -- COVERAGE=true bundle exec rspec
doppler run --config test -- bundle exec undercover --compare origin/main
```

If it reports uncovered lines → **do not declare success**. Options:
1. Add specs that cover the missing lines.
2. If the line is unreachable (e.g., Rails defensive branch), add `# :nocov:` with justification above.
3. Never silently bypass.

**Rule**: global 85% coverage (SimpleCov) and diff coverage (undercover) are **complementary**, not substitutes. Passing one doesn't excuse the other.

---

## Generic testing checklist

Before reporting a spec as "done":

- [ ] File in the correct location with correct `type:`.
- [ ] Happy path tested with **exact** values (`eq(100)`, not `be_present`).
- [ ] Error path tested (every exception the component can raise).
- [ ] Multi-tenant isolation tested (if applicable).
- [ ] Shared contexts reused, not reinvented.
- [ ] `instance_double` for non-model specs. `create(:factory)` only in model specs.
- [ ] Valid factories: `build(:factory, core_customer: core_customer).valid?`.
- [ ] Specs run in **isolation** (they can run alone without fixed ordering).
- [ ] `doppler run --config test -- rspec <spec>` passes green.
- [ ] `undercover --compare origin/main` with no uncovered lines in the diff.
