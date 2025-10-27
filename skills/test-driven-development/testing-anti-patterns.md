# Testing Anti-Patterns

**Load this reference when:** writing or changing tests, adding mocks, or tempted to add test-only methods to production code.

## Overview

Tests must verify real behavior, not mock behavior. Mocks are a means to isolate, not the thing being tested.

**Core principle:** Test what the code does, not what the mocks do.

**Following strict TDD prevents these anti-patterns.**

## The Iron Laws

```
1. NEVER test mock behavior
2. NEVER add test-only methods to production classes
3. NEVER mock without understanding dependencies
```

## Anti-Pattern 1: Testing Mock Behavior

**The violation:**
```ruby
# ❌ BAD: Testing that the stub was called, not actual behavior
RSpec.describe DashboardController, type: :controller do
  it "renders sidebar" do
    sidebar = double("Sidebar")
    allow(Sidebar).to receive(:new).and_return(sidebar)
    allow(sidebar).to receive(:render).and_return("<div>Mocked</div>")

    get :index

    expect(Sidebar).to have_received(:new)
  end
end
```

**Why this is wrong:**
- You're verifying the mock was called, not that the page works
- Test passes when mock is present, fails when it's not
- Tells you nothing about real behavior
- Violates BetterSpecs (testing implementation, not behavior)

**your human partner's correction:** "Are we testing the behavior of a mock?"

**The fix:**
```ruby
# ✅ GOOD: Test real behavior with request spec
RSpec.describe "Dashboard", type: :request do
  describe "GET /dashboard" do
    it "includes sidebar navigation" do
      get dashboard_path

      expect(response).to have_http_status(:success)
      expect(response.body).to include('<nav class="sidebar"')
    end
  end
end

# OR if testing controller logic in isolation:
RSpec.describe DashboardController, type: :controller do
  describe "GET #index" do
    it "assigns current user's deals" do
      user = create(:user)
      sign_in user
      deal = create(:deal, user: user)

      get :index

      expect(assigns(:deals)).to include(deal)
    end
  end
end
```

### Gate Function

```
BEFORE asserting on any mock/stub:
  Ask: "Am I testing real behavior or just that the mock was called?"

  IF testing mock was called:
    STOP - Delete the assertion or remove the mock

  Test real behavior instead (use request specs, real objects)
```

## Anti-Pattern 2: Test-Only Methods in Production

**The violation:**
```ruby
# ❌ BAD: reset_cache! only used in tests
class Deal < ApplicationRecord
  def reset_cache!  # Looks like production API!
    Rails.cache.delete("deal_#{id}_summary")
    reload
  end
end

# In spec
RSpec.describe Deal, type: :model do
  after { deal.reset_cache! }
end
```

**Why this is wrong:**
- Production model polluted with test-only code
- Dangerous if accidentally called in production
- Violates YAGNI and separation of concerns
- Confuses object lifecycle with test lifecycle

**The fix:**
```ruby
# ✅ GOOD: Test utilities handle test cleanup
# Deal model has no reset_cache! method

# spec/support/cache_helpers.rb
module CacheHelpers
  def clear_deal_cache(deal)
    Rails.cache.delete("deal_#{deal.id}_summary")
  end
end

RSpec.configure do |config|
  config.include CacheHelpers
end

# In spec
RSpec.describe Deal, type: :model do
  let(:deal) { create(:deal) }

  after { clear_deal_cache(deal) }

  # OR use database_cleaner/database truncation
  # to reset state between tests
end
```

### Gate Function

```
BEFORE adding any method to production model/class:
  Ask: "Is this only used by tests?"

  IF yes:
    STOP - Don't add it
    Put it in spec/support/ helpers instead

  Ask: "Does this class own this resource's lifecycle?"

  IF no:
    STOP - Wrong class for this method
```

## Anti-Pattern 3: Mocking Without Understanding

**The violation:**
```ruby
# ❌ BAD: Mock breaks test logic
RSpec.describe "Document processing" do
  it "prevents duplicate documents" do
    # Mock prevents database write that test depends on!
    allow(Document).to receive(:create!).and_return(double(id: 1))

    service = DocumentUploadService.new
    service.process(file)
    service.process(file)  # Should raise - but won't!

    # Test passes but doesn't actually test duplicate detection
  end
end
```

**Why this is wrong:**
- Mocked method had side effect test depended on (database write)
- Over-mocking to "be safe" breaks actual behavior
- Test passes for wrong reason or fails mysteriously
- Defeats purpose of integration testing

**The fix:**
```ruby
# ✅ GOOD: Test with real database, mock only external services
RSpec.describe DocumentUploadService, type: :service do
  subject(:service) { described_class.new }

  let(:file) { fixture_file_upload("sample.pdf") }

  before do
    # Mock only the slow external service (AWS Textract)
    allow(TextractClient).to receive(:start_document_analysis)
      .and_return(double(job_id: "job-123"))
  end

  describe "#process" do
    context "when processing duplicate file" do
      before { service.process(file) }

      it "raises error on duplicate" do
        expect { service.process(file) }
          .to raise_error(DocumentUploadService::DuplicateError)
      end
    end
  end
end
```

### Gate Function

```
BEFORE mocking any method in Rails:
  STOP - Don't mock yet

  1. Ask: "What side effects does the real method have?"
     (Database writes? Cache updates? File system changes?)
  2. Ask: "Does this test depend on any of those side effects?"
  3. Ask: "Do I fully understand what this test needs?"

  IF depends on side effects:
    Mock at lower level (external APIs, network calls)
    OR use test fixtures that preserve necessary behavior
    NOT the high-level method the test depends on

  IF unsure what test depends on:
    Run test with real implementation FIRST
    Use real database (DatabaseCleaner handles cleanup)
    THEN add minimal mocking at the right level (external services only)

  Red flags:
    - "I'll stub this to be safe"
    - "Database might be slow, better mock ActiveRecord"
    - Mocking without understanding the dependency chain
```

## Anti-Pattern 4: Incomplete Factory Definitions

**The violation:**
```ruby
# ❌ BAD: Partial factory - only fields you think you need
FactoryBot.define do
  factory :document do
    filename { "test.pdf" }
    content_type { "application/pdf" }
    # Missing: uploaded_by, organization, required associations
  end
end

# Later: breaks when code accesses document.uploaded_by or document.organization
```

**Why this is wrong:**
- **Partial factories hide structural assumptions** - You only defined fields you know about
- **Downstream code may depend on fields you didn't include** - Silent failures with nil
- **Tests pass but integration fails** - Factory incomplete, real records complete
- **False confidence** - Test proves nothing about real behavior
- **Violates BetterSpecs** - Test data should match production data structure

**The Iron Rule:** Define COMPLETE factory with ALL required associations and fields, not just fields your immediate test uses.

**The fix:**
```ruby
# ✅ GOOD: Complete factory matching production data structure
FactoryBot.define do
  factory :document do
    filename { "test.pdf" }
    content_type { "application/pdf" }
    file_size { 1024 }
    uploaded_at { Time.current }

    # Required associations
    association :uploaded_by, factory: :user
    association :organization
    association :deal

    # State
    status { :pending }

    # Traits for variations
    trait :processed do
      status { :processed }
      processed_at { Time.current }
    end

    trait :with_classification do
      after(:create) do |document|
        create(:document_classification, document: document)
      end
    end
  end
end
```

### Gate Function

```
BEFORE creating factories:
  Check: "What fields and associations does the real model require?"

  Actions:
    1. Examine model validations and associations
    2. Include ALL required fields and associations
    3. Add traits for common variations (don't create multiple factories)
    4. Verify factory can create valid record: build(:document).valid?

  Critical:
    If you're creating a factory, it must create VALID production-like records
    Partial factories fail silently when code depends on omitted fields/associations

  Rails-specific:
    - Include all belongs_to associations (required by default in Rails 5+)
    - Include all presence validations
    - Use traits for state variations, not separate factories
```

## Anti-Pattern 5: Integration Tests as Afterthought

**The violation:**
```
✅ Implementation complete
❌ No tests written
"Ready for testing"
```

**Why this is wrong:**
- Testing is part of implementation, not optional follow-up
- TDD would have caught this
- Can't claim complete without tests

**The fix:**
```
TDD cycle:
1. Write failing test
2. Implement to pass
3. Refactor
4. THEN claim complete
```

## When Mocks Become Too Complex

**Warning signs:**
- Mock setup longer than test logic
- Mocking everything to make test pass
- Mocks missing methods real components have
- Test breaks when mock changes

**your human partner's question:** "Do we need to be using a mock here?"

**Consider:** Integration tests with real components often simpler than complex mocks

## TDD Prevents These Anti-Patterns

**Why TDD helps:**
1. **Write test first** → Forces you to think about what you're actually testing
2. **Watch it fail** → Confirms test tests real behavior, not mocks
3. **Minimal implementation** → No test-only methods creep in
4. **Real dependencies** → You see what the test actually needs before mocking

**If you're testing mock behavior, you violated TDD** - you added mocks without watching test fail against real code first.

## Quick Reference

| Anti-Pattern | Fix |
|--------------|-----|
| Assert on mock elements | Test real component or unmock it |
| Test-only methods in production | Move to test utilities |
| Mock without understanding | Understand dependencies first, mock minimally |
| Incomplete mocks | Mirror real API completely |
| Tests as afterthought | TDD - tests first |
| Over-complex mocks | Consider integration tests |

## Red Flags

- Assertion checks for `*-mock` test IDs
- Methods only called in test files
- Mock setup is >50% of test
- Test fails when you remove mock
- Can't explain why mock is needed
- Mocking "just to be safe"

## The Bottom Line

**Mocks are tools to isolate, not things to test.**

If TDD reveals you're testing mock behavior, you've gone wrong.

Fix: Test real behavior or question why you're mocking at all.
