# Services — Query, Action, LLM Tool

Three distinct Service Object patterns in core-web-api. Each has its place and clear signals to know which to apply.

---

## Quick decision tree

```
What do I need to do?

├── Read data without mutating (listings, searches, reports)
│   → Query Service

├── Orchestrate N operations (multiple recorders + emails + jobs)
│   → Action Service

├── Expose something to Kathy's LLM (read or write with confirmation)
│   → LLM Tool (inherits RubyLLM::Tool)

└── One CRUD operation + one event
    → Recorder (NOT a service, see recorders.md)
```

**Rule**: if the operation fits in a Recorder, use a Recorder. Services are for what Recorders don't cover.

---

## Location and naming

```
app/services/
├── <module>/
│   ├── <resources>/
│   │   └── <operation>_service.rb        # Query / Action
│   └── ...
└── kathy/
    └── llm/
        ├── tools/
        │   ├── <name>.rb                 # Top-level LLM Tool
        │   └── <module>/
        │       └── <name>.rb             # LLM Tool per module
        ├── helpers/
        │   └── <shared>.rb               # Reusable concerns
        └── chat_service.rb
```

---

## 1. Query Service — read-only

Reads data without mutation. Returns a formatted string (Toon-encoded for LLMs) or a collection ready to serialize.

### Basic pattern

```ruby
module Kathy
  module Llm
    module Tools
      class SearchFaqs < RubyLLM::Tool
        include Helpers::SentryInstrumentation

        description "Searches the business knowledge base for FAQ-style information..."

        def execute
          faqs = Kathy::Faq.active.select(:question, :answer).limit(Kathy::Faq::MAX_ACTIVE_QUESTIONS_PER_CUSTOMER)
          return format_empty_response if faqs.empty?

          format_faqs_response(faqs)
        rescue StandardError => e
          capture_tool_error(e)
          format_error_response
        end

        private

        def format_faqs_response(faqs)
          data = { "faqs" => faqs.map { |faq| { "q" => faq.question, "a" => faq.answer } } }
          Toon.encode(data)
        end

        def format_empty_response
          "No FAQs available at this time."
        end

        def format_error_response
          "Error searching the knowledge base."
        end
      end
    end
  end
end
```

### Observations
- **Mutates nothing**: pure SELECT + formatting.
- **Dedicated responses for 3 states**: success, empty, error. Each with its own private method.
- **User-friendly messages** on error — never expose stack traces to the client/LLM.
- **`capture_tool_error(e)`** sends the error to Sentry with context, but returns something readable.

---

## 2. Action Service — orchestration

When an operation requires **multiple recorders + side-effects** (emails, jobs, complex calculations). An Action Service **invokes recorders**, doesn't duplicate their logic.

### Typical structure

```ruby
module Kommerce
  module Orders
    class CreateFromBookingService
      def initialize(booking:, payment_params:)
        @booking = booking
        @payment_params = payment_params
      end

      def execute
        order = create_order
        attach_payment(order)
        enqueue_notifications(order)
        order
      rescue ActiveRecord::RecordInvalid => e
        Rails.logger.error("[Kommerce::CreateFromBooking] #{e.record.errors.full_messages.join(', ')}")
        raise
      end

      private

      attr_reader :booking, :payment_params

      def create_order
        ::Kommerce::Orders::CreateRecorder.new(
          params: { kalendar_booking: booking, total: booking.total }
        ).record
      end

      def attach_payment(order)
        ::Kommerce::Payments::CreateRecorder.new(
          params: payment_params.merge(kommerce_order: order)
        ).record
      end

      def enqueue_notifications(order)
        ::Kommerce::OrderConfirmationJob.perform_later(order.id)
      end
    end
  end
end
```

### Action Service rules
- **Each step is atomic**: one private method per operation.
- **Delegates to Recorders** for all persistence. Do not call `Model.create!` directly in the service.
- **Jobs receive IDs**, never objects.
- **Explicit transaction** if the steps must be atomic: wrap in `ActiveRecord::Base.transaction do ... end`.
- **Specific rescue first**, then `raise` if it can't resolve it — let the controller/job handle it.
- **Logging with context**: `[Module::ServiceName]` as a prefix for the message.

---

## 3. LLM Tool — for Kathy (RubyLLM)

Inherits from `RubyLLM::Tool`. Defines `description`, `params`, and `execute`. It's the bridge between the LLM and business logic.

### 3a. Read-only LLM Tool (like a Query Service)

See `SearchFaqs` above. Can use `Current.user` for per-user scoping, and `Toon.encode` for formatting.

### 3b. Mutating LLM Tool — **confirmation flow required**

Two phases:
1. `confirmed: false` → return `halt(summary)` with a preview of the action.
2. `confirmed: true` → execute (delegating to a Recorder).

**Real example**: `Kathy::Llm::Tools::Kalendar::CreateBooking`:

```ruby
module Kathy
  module Llm
    module Tools
      module Kalendar
        class CreateBooking < RubyLLM::Tool
          include ::Kalendar::Bookings::Concerns::TotalCalculator
          include Helpers::BookingFormatter
          include Helpers::SentryInstrumentation

          description "Creates a new booking for a resource. Always present the booking details " \
                      "(resource name, date, time, duration, and estimated total) and ask for " \
                      "explicit confirmation BEFORE calling this tool with confirmed: true."

          params do
            integer :resource_id, description: "ID of the resource/service to book"
            string :start_at, description: "Start date and time in ISO 8601 format: YYYY-MM-DDTHH:MM:SS"
            string :notes, description: "Additional notes for the booking", required: false
            boolean :confirmed, description: "Whether the customer explicitly confirmed the booking details"
          end

          def execute(resource_id:, start_at:, confirmed:, notes: nil)
            resource = ::Kalendar::Resource.active.find(resource_id)
            start_datetime = Time.zone.parse(start_at)
            return "Invalid date format. Please use ISO 8601 format: YYYY-MM-DDTHH:MM:SS" unless start_datetime

            end_datetime = start_datetime + (resource.duration_in_minutes * 60)

            if confirmed
              create_booking(resource, start_datetime, end_datetime, notes)
            else
              format_confirmation_request(resource, start_datetime, end_datetime)
            end
          rescue ActiveRecord::RecordNotFound
            "Resource not found. Please check the resource ID and try again."
          rescue ActiveRecord::RecordInvalid => e
            "Unable to create booking: #{e.record.errors.full_messages.join(', ')}"
          rescue ActiveRecord::StatementInvalid => e
            capture_tool_error(e, tool_params: { resource_id: resource_id, start_at: start_at })
            "Database error while creating booking. Please try again."
          rescue StandardError => e
            capture_tool_error(e, tool_params: { resource_id: resource_id, start_at: start_at })
            "Something went wrong while processing your booking. Please try again later."
          end

          private

          def format_confirmation_request(resource, start_datetime, end_datetime)
            # ... builds preview with set_total, returns halt(summary)
            halt(summary)
          end

          def create_booking(resource, start_datetime, end_datetime, notes)
            params = {
              kalendar_resource_id: resource.id,
              start_at: start_datetime,
              end_at: end_datetime,
              booked_by_id: Current.user.id,
              booked_by_assisted: true,
              notes: notes
            }

            recorder = ::Kalendar::Bookings::CreateRecorder.new(params: params)
            booking = recorder.record

            format_booking_confirmation(booking, resource)
          end
        end
      end
    end
  end
end
```

### Critical points of the confirmation flow
- **`halt(summary)`** stops tool execution and returns the summary to the LLM. The LLM presents the preview to the user and waits for explicit confirmation.
- **`confirmed: true`** is only received when the user confirmed affirmatively. The tool executes the recorder and returns the final result (Toon-encoded).
- **NEVER call `create!` directly** in the tool — always delegate to the corresponding Recorder.
- **`booked_by_assisted: true`** (or equivalent flag) marks the action as LLM-mediated — useful for auditing and metrics.

### Rescue cascade for LLM Tools
Ordered from specific to generic:

1. `ActiveRecord::RecordNotFound` → user-friendly message with instructions.
2. `ActiveRecord::RecordInvalid` → include `errors.full_messages` (safe, doesn't expose internals).
3. `ActiveRecord::StatementInvalid` → log to Sentry + generic message.
4. `StandardError` → log to Sentry + generic message. **Never** `raise` to the LLM.

---

## Toon encoding

Compact (JSON-like) format for LLM tool responses. Reduces tokens compared to standard JSON.

```ruby
data = { bookings: [...] }
Toon.encode(data)   # returns compact string
```

**Rule**: every successful LLM tool response that contains structured data → `Toon.encode`. Simple messages (errors, "no data") → plain string.

---

## Shared helpers/concerns

Live in `app/services/kathy/llm/helpers/`:

| Concern | Use |
|---|---|
| `Helpers::SentryInstrumentation` | `capture_tool_error(e, tool_params:)` |
| `Helpers::BookingFormatter` | `format_total`, `format_status`, `format_booking_summary` |
| `Kalendar::Bookings::Concerns::TotalCalculator` | `set_total` — calculates booking total in progress |

**Rule**: create a helper ONLY when the logic repeats across ≥2 tools. Not preemptively.

---

## Testing — **no DB contact**

Hard repo rule: **service specs use mocks**. Never `create(:factory)` in a service spec.

### Query Service spec

```ruby
require "rails_helper"

RSpec.describe Kathy::Llm::Tools::SearchFaqs do
  subject(:tool) { described_class.new }

  let(:faq_double) { instance_double(::Kathy::Faq, question: "Q?", answer: "A.") }
  let(:relation) { instance_double(ActiveRecord::Relation) }

  describe "#execute" do
    it "returns Toon-encoded FAQs when results exist" do
      expect(::Kathy::Faq).to receive(:active).and_return(relation)
      expect(relation).to receive(:select).with(:question, :answer).and_return(relation)
      expect(relation).to receive(:limit).and_return([faq_double])
      expect(relation).to receive(:empty?).and_return(false) if relation.respond_to?(:empty?)

      result = tool.execute

      expect(result).to be_a(String)
      expect(result).to include("Q?")
    end

    it "returns empty message when no FAQs" do
      expect(::Kathy::Faq).to receive(:active).and_return(relation)
      allow(relation).to receive(:select).and_return(relation)
      allow(relation).to receive(:limit).and_return([])

      expect(tool.execute).to eq("No FAQs available at this time.")
    end

    it "returns error message on StandardError" do
      expect(::Kathy::Faq).to receive(:active).and_raise(StandardError.new("db down"))

      expect(tool.execute).to eq("Error searching the knowledge base.")
    end
  end
end
```

### LLM Tool with confirmation — spec with two contexts

```ruby
RSpec.describe Kathy::Llm::Tools::Kalendar::CreateBooking do
  subject(:tool) { described_class.new }

  let(:resource) { instance_double(::Kalendar::Resource, id: 1, name: "Service", duration_in_minutes: 60) }
  let(:user)     { instance_double(::Customer::User, id: 42) }
  let(:booking)  { instance_double(::Kalendar::Booking, id: 99, start_at: Time.zone.parse("2026-04-20T10:00:00"), end_at: Time.zone.parse("2026-04-20T11:00:00"), total: 50.0, status: "pending") }

  before do
    allow(::Kalendar::Resource).to receive_message_chain(:active, :find).with(1).and_return(resource)
    allow(Current).to receive(:user).and_return(user)
  end

  describe "#execute" do
    context "when confirmed: false" do
      it "returns halt summary and does not create booking" do
        expect(::Kalendar::Bookings::CreateRecorder).not_to receive(:new)

        result = tool.execute(resource_id: 1, start_at: "2026-04-20T10:00:00", confirmed: false)

        expect(result).to be_a(RubyLLM::Tool::Halt)
        expect(result.content).to include("Service")
      end
    end

    context "when confirmed: true" do
      let(:recorder) { instance_double(::Kalendar::Bookings::CreateRecorder, record: booking) }

      it "delegates to recorder and returns Toon-encoded confirmation" do
        expect(::Kalendar::Bookings::CreateRecorder).to receive(:new)
          .with(params: hash_including(kalendar_resource_id: 1, booked_by_id: 42, booked_by_assisted: true))
          .and_return(recorder)

        result = tool.execute(resource_id: 1, start_at: "2026-04-20T10:00:00", confirmed: true)

        expect(result).to be_a(String)
        expect(result).to include("Service")
      end
    end

    context "when resource not found" do
      before { allow(::Kalendar::Resource).to receive_message_chain(:active, :find).and_raise(ActiveRecord::RecordNotFound) }

      it "returns user-friendly message" do
        expect(tool.execute(resource_id: 999, start_at: "2026-04-20T10:00:00", confirmed: true))
          .to eq("Resource not found. Please check the resource ID and try again.")
      end
    end

    context "when date format is invalid" do
      it "returns format error" do
        expect(tool.execute(resource_id: 1, start_at: "not-a-date", confirmed: true))
          .to eq("Invalid date format. Please use ISO 8601 format: YYYY-MM-DDTHH:MM:SS")
      end
    end
  end
end
```

### Service testing rules
- `instance_double` for every DB resource (`Resource`, `User`, `Booking`).
- `instance_double(::Kalendar::Bookings::CreateRecorder, record: booking_double)` to mock internal recorders.
- `allow(Current).to receive(:user).and_return(user_double)` to simulate user context.
- Test the 3 states: happy path, each expected error, edge cases (invalid input).
- Test both sides of the confirmation flow: `confirmed: false` (preview) and `confirmed: true` (execution).
- **Never** `create(:booking)` or `build(:resource)` inside a service spec.

---

## When it's NOT a Service

- **Single CRUD operation + one event** → it's a `Recorder`.
- **View logic over a model** → it's a `Decorator`.
- **Async work** → it's a `Job`.
- **Email** → it's a `Mailer`.

**Signal**: if your service has a single private method that does `Model.create!`, delete the service and use a Recorder.

---

## Checklist for creating a Service

- [ ] Identified the correct type (Query / Action / LLM Tool) using the decision tree.
- [ ] Correct location in `app/services/<module>/...`.
- [ ] **Action Service** delegates to Recorders, doesn't do `Model.create!` directly.
- [ ] **LLM Tool with mutation** implements confirmation flow (`halt(summary)` → `confirmed: true`).
- [ ] Errors rescued from specific to generic, with user-friendly messages.
- [ ] `capture_tool_error(e)` for unexpected errors in LLM Tools.
- [ ] Structured responses use `Toon.encode`.
- [ ] Spec with `instance_double` — **no** `create(:factory)`.
- [ ] Spec covers happy path, each expected error, edge cases.
- [ ] LLM Tool spec with confirmation covers both states (`confirmed: false` and `confirmed: true`).
