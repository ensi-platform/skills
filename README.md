# Ensi Skills

A repository of skills for AI assistants working with Ensi projects. Contains specialized instructions for development in the Ensi ecosystem.

## Installation

### Basic Installation

```bash
# Install all skills
npx skills add ensi-platform/skills

# Install specific skills
npx skills add ensi-platform/skills --skill ensi-openapi --skill ensi-code-style

# List available skills
npx skills add ensi-platform/skills --list

# Install globally (available in all projects)
npx skills add ensi-platform/skills -g

# Install for specific agents
npx skills add ensi-platform/skills -a claude-code -a opencode
```

### Usage Examples

```bash
# Install all skills for OpenCode
npx skills add ensi-platform/skills --all -a opencode

# Install only OpenAPI skill globally
npx skills add ensi-platform/skills --skill ensi-openapi -g -y

# Interactive installation
npx skills add ensi-platform/skills
```

## Available Skills

### Ensi Backend Development

| Skill | Description |
|-------|-------------|
| **ensi-openapi** | Work with OpenAPI specifications in Ensi services. Create endpoints, schemas, enums, work with yaml files in `public/api-docs/`. |
| **ensi-code-style** | Enforce PHP and Laravel code style according to Ensi guidelines. Create classes, set up validation, routes, migrations. |
| **ensi-models** | Work with Eloquent models in Ensi projects. Create models, factories, relationships, scopes, migrations. |
| **ensi-tests** | Work with Pest PHP tests in Ensi services. Create tests for API endpoints, components, unit tests. |
| **ensi-api-design** | Apply Ensi API Design Guide principles when designing REST API endpoints. |
| **ensi-meta** | Create and modify meta endpoints in Ensi services. Work with Field classes, ModelMetaResource, EnumInfo. |
| **ensi-query-builder** | Create and modify Query classes for spatie/laravel-query-builder. Work with filters, sorting, includes. |
| **ensi-kafka** | Work with Kafka in Ensi Laravel services. Create producers, consumers, observers, payloads. |

### DevOps

| Skill | Description |
|-------|-------------|
| **ansible-component** | Work with DevOps components in IaC repository. Create ansible playbooks for kubernetes, server applications, configs, secrets. |


## Agent Commands

This repository also contains agent command definitions that are not installed via `npx skills`. These are special configuration files for specific agent workflows:

| Command | Description |
|---------|-------------|
| **propose** | Detail task and create proposal - transforms brief feature requests into comprehensive proposal documents |
| **plan** | Break down feature proposal into tasks - creates detailed, actionable task lists from proposals |
| **task-coordinator** | Task execution coordinator - stores feature context and delegates tasks to subagents sequentially |

These files are located in `commands/` and `agents/` directories and are used directly by the agent configuration, not through the skills CLI.

## Development Workflow

This repository provides a structured workflow for feature development using agent commands:

### Step 1: Create Feature Proposal

```
/propose <brief feature description>
```

The agent analyzes your request and creates a comprehensive proposal document:
- Creates `{AI_STABLE_CONTEXT}/features/{feature-name}/proposal.md`
- Expands requirements through targeted questions
- Documents functional and non-functional requirements
- Identifies edge cases and integration points
- Records references to existing code patterns

### Step 2: Break Down into Tasks

```
/plan <feature-name>
```

The agent breaks down the proposal into actionable tasks:
- Creates `{AI_STABLE_CONTEXT}/features/{feature-name}/tasks.md`
- Generates 5-20 detailed, self-contained tasks
- Each task has specific file paths and acceptance criteria
- Groups related work (e.g., migration + model, action + tests)

### Step 3: Execute Tasks with Coordinator

```
@task-coordinator <feature-name>
```

The task-coordinator agent manages execution:
- Reads both `proposal.md` and `tasks.md`
- Initializes a todo list with all tasks
- Stores feature context in memory
- For each task:
  - Delegates to a subagent with full context
  - Tracks completed changes for subsequent tasks
  - Asks for confirmation before marking complete
  - Suggests commit messages

### Step 4: Iterate and Refine

After each task completion:
- Review the changes
- Accept or request corrections
- Proceed to the next task with accumulated context

### Example Workflow

```bash
# Session 1: Create proposal
/propose Add OAuth2 authentication for users
# → Creates features/user-auth-oauth2/proposal.md

# Session 2: Create task breakdown
/plan user-auth-oauth2
# → Creates features/user-auth-oauth2/tasks.md with 12 tasks

# Session 3: Execute tasks
task-coordinator user-auth-oauth2
# → Waits for "execute next task" command
execute next task
# → Delegates to subagent, shows changes, asks for confirmation
yes
# → Updates tasks.md, suggests commit, waits for next command
execute next task
# ... continues until all tasks are complete
```

### Benefits

- **Incremental execution**: Work on one task at a time with full context
- **Tracking**: Automatic todo list and task status management
- **Review points**: Built-in confirmation after each task
- **Context awareness**: Subagents know about previous changes in the same feature
- **Traceability**: All proposals and tasks are saved for reference

## Supported Agents

Skills are tested with the following AI assistants:

- **OpenCode** (`opencode`)

## Documentation

- [Agent Skills Specification](https://agentskills.io)
- [Skills CLI Documentation](https://github.com/vercel-labs/skills)
- [OpenCode Skills Documentation](https://opencode.ai/docs/skills)

## Development

### Adding a new skill

1. Create a directory in `skills/`
2. Add `SKILL.md` with frontmatter
3. Ensure `name` is unique
4. Add detailed instructions in the document body

### Updating a skill

1. Edit the corresponding `SKILL.md`
2. Users can update skills with the command:
   ```bash
   npx skills update
   ```
