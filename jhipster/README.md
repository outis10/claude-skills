# JHipster Development Skills

Custom skills for Claude Code to assist with JHipster project development.

## Skills

1. **00-kalitron-suite-master.md** - Master skill for KALITRON branding and project coordination
   - KALITRON banner management
   - Project initialization
   - Cross-cutting concerns

2. **01-backend-development.md** - Spring Boot backend patterns
   - Entity creation with JPA
   - Service/Repository layers
   - REST API controllers
   - Liquibase migrations
   - Security configuration
   - DTOs and MapStruct mappers

3. **02-react-frontend.md** - React frontend patterns
   - TypeScript entity models
   - Redux Toolkit state management
   - CRUD view components
   - Form validation
   - Routing configuration
   - WebSocket integration

## Installation

Copy the `.md` files to your Claude Code skills directory:

```bash
# Copy all skills to the Claude Code custom skills location
cp jhipster/*.md ~/.claude/skills/
```

Or reference them in your project's `.claude/settings.json`:

```json
{
  "skills": [
    "jhipster/00-kalitron-suite-master.md",
    "jhipster/01-backend-development.md",
    "jhipster/02-react-frontend.md"
  ]
}
```

## Usage

Claude will automatically use the appropriate skill based on context:

- "agrega el banner kalitron" -> Master skill (banner management)
- "crea entidad jhipster" -> Backend Development skill
- "genera componente react" -> React Frontend skill
- "setup jhipster completo" -> Uses all three skills
