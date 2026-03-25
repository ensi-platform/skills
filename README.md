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
