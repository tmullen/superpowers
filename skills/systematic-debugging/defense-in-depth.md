# Defense-in-Depth Validation

## Overview

When you fix a bug caused by invalid data, adding validation at one place feels sufficient. But that single check can be bypassed by different code paths, refactoring, or mocks.

**Core principle:** Validate at EVERY layer data passes through. Make the bug structurally impossible.

## Why Multiple Layers

Single validation: "We fixed the bug"
Multiple layers: "We made the bug impossible"

Different layers catch different cases:
- Entry validation catches most bugs
- Business logic catches edge cases
- Environment guards prevent context-specific dangers
- Debug logging helps when other layers fail

## The Four Layers

### Layer 1: Entry Point Validation
**Purpose:** Reject obviously invalid input at API boundary

```typescript
function createProject(name: string, workingDirectory: string) {
  if (!workingDirectory || workingDirectory.trim() === '') {
    throw new Error('workingDirectory cannot be empty');
  }
  if (!existsSync(workingDirectory)) {
    throw new Error(`workingDirectory does not exist: ${workingDirectory}`);
  }
  if (!statSync(workingDirectory).isDirectory()) {
    throw new Error(`workingDirectory is not a directory: ${workingDirectory}`);
  }
  // ... proceed
}
```

### Layer 2: Business Logic Validation
**Purpose:** Ensure data makes sense for this operation

```typescript
function initializeWorkspace(projectDir: string, sessionId: string) {
  if (!projectDir) {
    throw new Error('projectDir required for workspace initialization');
  }
  // ... proceed
}
```

### Layer 3: Environment Guards
**Purpose:** Prevent dangerous operations in specific contexts

```typescript
async function gitInit(directory: string) {
  // In tests, refuse git init outside temp directories
  if (process.env.NODE_ENV === 'test') {
    const normalized = normalize(resolve(directory));
    const tmpDir = normalize(resolve(tmpdir()));

    if (!normalized.startsWith(tmpDir)) {
      throw new Error(
        `Refusing git init outside temp dir during tests: ${directory}`
      );
    }
  }
  // ... proceed
}
```

### Layer 4: Debug Instrumentation
**Purpose:** Capture context for forensics

```typescript
async function gitInit(directory: string) {
  const stack = new Error().stack;
  logger.debug('About to git init', {
    directory,
    cwd: process.cwd(),
    stack,
  });
  // ... proceed
}
```

## Rails Four-Layer Pattern

### Layer 1: Strong Parameters (Controller Entry Point)
**Purpose:** Reject invalid input at HTTP boundary

```ruby
# app/controllers/projects_controller.rb
class ProjectsController < ApplicationController
  def create
    # Entry point validation - reject bad params immediately
    @project = Project.create!(project_params)
    redirect_to @project
  rescue ActionController::ParameterMissing => e
    render json: { error: "Missing required parameter: #{e.param}" }, status: :bad_request
  end

  private

  def project_params
    params.require(:project).permit(:name, :working_directory, :organization_id)
  end
end
```

### Layer 2: Model Validations (Business Logic)
**Purpose:** Ensure data makes sense for domain model

```ruby
# app/models/project.rb
class Project < ApplicationRecord
  belongs_to :organization

  # Business logic validation
  validates :name, presence: true, length: { maximum: 255 }
  validates :working_directory, presence: true
  validate :directory_must_exist
  validate :directory_must_be_writable

  private

  def directory_must_exist
    return if working_directory.blank?
    return if Dir.exist?(working_directory)

    errors.add(:working_directory, "does not exist: #{working_directory}")
  end

  def directory_must_be_writable
    return if working_directory.blank?
    return unless Dir.exist?(working_directory)
    return if File.writable?(working_directory)

    errors.add(:working_directory, "is not writable")
  end
end
```

### Layer 3: Database Constraints (Data Integrity)
**Purpose:** Prevent invalid data at storage layer

```ruby
# db/migrate/20250127_create_projects.rb
class CreateProjects < ActiveRecord::Migration[8.0]
  def change
    create_table :projects do |t|
      t.string :name, null: false
      t.string :working_directory, null: false
      t.references :organization, null: false, foreign_key: true

      t.timestamps
    end

    # Database-level constraints ensure data integrity
    add_index :projects, :working_directory
    add_check_constraint :projects, "length(name) > 0", name: "name_not_empty"
    add_check_constraint :projects, "length(working_directory) > 0", name: "directory_not_empty"
  end
end
```

### Layer 4: Service Object Validation (Workflow Guard)
**Purpose:** Validate context and workflow requirements

```ruby
# app/services/project_initializer.rb
class ProjectInitializer
  class InvalidProjectError < StandardError; end

  def initialize(project)
    # Service-level validation for workflow
    raise ArgumentError, "project required" if project.nil?
    raise ArgumentError, "project must be persisted" unless project.persisted?
    raise InvalidProjectError, "project directory blank" if project.working_directory.blank?

    # Debug instrumentation
    Rails.logger.debug do
      "ProjectInitializer called: project_id=#{project.id}, " \
      "directory=#{project.working_directory}, " \
      "caller=#{caller[0..2].join(' <- ')}"
    end

    @project = project
  end

  def call
    verify_directory_state!
    initialize_workspace
    @project
  end

  private

  def verify_directory_state!
    # Additional runtime checks
    unless Dir.exist?(@project.working_directory)
      raise InvalidProjectError,
        "Directory disappeared: #{@project.working_directory}"
    end

    # Environment-specific guard (like Layer 3 TypeScript example)
    if Rails.env.test? && !@project.working_directory.start_with?(Dir.tmpdir)
      raise InvalidProjectError,
        "Refusing to initialize outside temp directory in tests: #{@project.working_directory}"
    end
  end

  def initialize_workspace
    # ... actual initialization
  end
end
```

## Applying the Pattern

When you find a bug:

1. **Trace the data flow** - Where does bad value originate? Where used?
2. **Map all checkpoints** - List every point data passes through
3. **Add validation at each layer** - Entry, business, environment, debug
4. **Test each layer** - Try to bypass layer 1, verify layer 2 catches it

## Example from Session

Bug: Empty `projectDir` caused `git init` in source code

**Data flow:**
1. Test setup → empty string
2. `Project.create(name, '')`
3. `WorkspaceManager.createWorkspace('')`
4. `git init` runs in `process.cwd()`

**Four layers added:**
- Layer 1: `Project.create()` validates not empty/exists/writable
- Layer 2: `WorkspaceManager` validates projectDir not empty
- Layer 3: `WorktreeManager` refuses git init outside tmpdir in tests
- Layer 4: Stack trace logging before git init

**Result:** All 1847 tests passed, bug impossible to reproduce

## Key Insight

All four layers were necessary. During testing, each layer caught bugs the others missed:
- Different code paths bypassed entry validation
- Mocks bypassed business logic checks
- Edge cases on different platforms needed environment guards
- Debug logging identified structural misuse

**Don't stop at one validation point.** Add checks at every layer.
