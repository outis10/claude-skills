# JHipster Development Suite - KALITRON

## Purpose
Central skill for JHipster project development. This skill handles KALITRON branding and coordinates with specialized skills for backend and frontend development.

## Related Skills
This suite works in conjunction with:
- **JHipster Backend Development Skill**: Handles Spring Boot, entities, services, repositories
- **JHipster React Frontend Development Skill**: Handles React components, state management, routing

## Trigger Phrases
- "agrega el banner kalitron" → This skill (banner management)
- "crea entidad jhipster" → Backend Development Skill
- "genera componente react" → React Frontend Skill
- "setup jhipster completo" → Uses all three skills

## Current Capabilities
### 1. KALITRON Banner Management

## Integration Notes**Description**: Adds or updates the KALITRON branded banner in JHipster Spring Boot projects.

**Implementation Steps**:

1. Use `bash_tool` to verify project structure:
```bash
   ls -la src/main/resources/
```

2. If `src/main/resources/` doesn't exist, inform user this doesn't appear to be a JHipster/Spring Boot project

3. Use `create_file` tool to create/overwrite `src/main/resources/banner.txt` with the exact content below

4. Confirm to user that the banner has been added/updated successfully

**Banner Content**:
```
${AnsiColor.GREEN}██╗  ██╗${AnsiColor.CYAN} █████╗   ██╗     ██╗ ████████╗ ██████╗   ██████╗  ███╗    ██╗
${AnsiColor.GREEN}██║ ██╔╝${AnsiColor.CYAN} ██╔══██╗ ██║     ██║ ╚══██╔══╝ ██╔══██╗ ██╔═══██╗ ████╗   ██║
${AnsiColor.GREEN}█████╔╝ ${AnsiColor.CYAN} ███████║ ██║     ██║    ██║    ██████╔╝ ██║   ██║ ██╔██╗  ██║
${AnsiColor.GREEN}██╔═██╗ ${AnsiColor.CYAN} ██╔══██║ ██║     ██║    ██║    ██╔══██╗ ██║   ██║ ██║╚██╗ ██║
${AnsiColor.GREEN}██║  ██╗${AnsiColor.CYAN} ██║  ██║ ██████╗ ██║    ██║    ██║  ██║ ╚██████╔╝ ██║ ╚████║
${AnsiColor.GREEN}╚═╝  ╚═╝${AnsiColor.CYAN} ╚═╝  ╚═╝ ╚═════╝ ╚═╝    ╚═╝    ╚═╝  ╚═╝  ╚═════╝  ╚═╝  ╚═══╝
${AnsiColor.BRIGHT_BLUE}:: KALITRON 🚀  :: Running Spring Boot ${spring-boot.version} :: Startup profile(s) ${spring.profiles.active} ::${AnsiColor.DEFAULT}
```

**Technical Notes**:
- Spring Boot automatically loads `banner.txt` from `src/main/resources/` on startup
- ANSI color codes work in most modern terminals
- Placeholders `${spring-boot.version}` and `${spring.profiles.active}` are replaced by Spring Boot at runtime
- File must be exactly at `src/main/resources/banner.txt`

- When user requests backend features (entities, services, APIs), defer to Backend Development Skill
- When user requests frontend features (components, pages, routing), defer to React Frontend Skill
- This skill focuses on: branding, project initialization, cross-cutting concerns
