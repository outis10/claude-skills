# JHipster Development Skills

Custom skills for Claude Code to assist with JHipster project development.

## Structure

```
jhipster/
├── kalitron-suite/
│   └── SKILL.md              # KALITRON branding and project coordination
├── jhipster-backend/
│   └── SKILL.md              # Spring Boot backend patterns
├── jhipster-react-frontend/
│   ├── SKILL.md              # React frontend patterns
│   └── references/
│       └── component-examples.md  # Detailed component examples
└── README.md
```

## Skills

1. **kalitron-suite** - KALITRON banner management, project initialization, cross-cutting concerns
2. **jhipster-backend** - Entity creation, service/repository layers, REST APIs, Liquibase, security, DTOs/MapStruct
3. **jhipster-react-frontend** - TypeScript models, Redux Toolkit, CRUD views, forms, routing, WebSocket, i18n

## Installation

Copy the skill directories to your Claude Code skills location:

```bash
cp -r jhipster/kalitron-suite ~/.claude/skills/
cp -r jhipster/jhipster-backend ~/.claude/skills/
cp -r jhipster/jhipster-react-frontend ~/.claude/skills/
```

Or place them in your project's `.claude/skills/` directory.

## Usage

Claude automatically selects the appropriate skill based on context:

- "agrega el banner kalitron" -> kalitron-suite
- "crea entidad jhipster" -> jhipster-backend
- "genera componente react" -> jhipster-react-frontend
- "setup jhipster completo" -> Uses all three skills


Opcion 1: Clonar el repo y hacer symlink**

En cada maquina:
```bash
git clone https://github.com/outis10/claude-skills.git ~/claude-skills
mkdir -p ~/.claude/skills

ln -s ~/claude-skills/jhipster/kalitron-suite ~/.claude/skills/
ln -s ~/claude-skills/jhipster/jhipster-backend ~/.claude/skills/
ln -s ~/claude-skills/jhipster/jhipster-react-frontend ~/.claude/skills/
```

Asi cuando hagas `git pull` se actualizan automaticamente.
