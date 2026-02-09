---
name: kalitron-suite
description: "Central skill for JHipster KALITRON project setup and branding. Use when the user mentions: KALITRON banner, JHipster project initialization, Spring Boot banner customization, or 'agrega el banner kalitron'. Handles KALITRON branding and coordinates with jhipster-backend and jhipster-react-frontend skills for specialized tasks."
---

# JHipster Development Suite - KALITRON

## KALITRON Banner Management

Adds or updates the KALITRON branded banner in JHipster Spring Boot projects.

### Steps

1. Verify project structure with `ls -la src/main/resources/`
2. If `src/main/resources/` doesn't exist, inform user this doesn't appear to be a JHipster/Spring Boot project
3. Create/overwrite `src/main/resources/banner.txt` with the banner content below
4. Confirm to user that the banner has been added/updated successfully

### Banner Content

```
${AnsiColor.GREEN}██╗  ██╗${AnsiColor.CYAN} █████╗   ██╗     ██╗ ████████╗ ██████╗   ██████╗  ███╗    ██╗
${AnsiColor.GREEN}██║ ██╔╝${AnsiColor.CYAN} ██╔══██╗ ██║     ██║ ╚══██╔══╝ ██╔══██╗ ██╔═══██╗ ████╗   ██║
${AnsiColor.GREEN}█████╔╝ ${AnsiColor.CYAN} ███████║ ██║     ██║    ██║    ██████╔╝ ██║   ██║ ██╔██╗  ██║
${AnsiColor.GREEN}██╔═██╗ ${AnsiColor.CYAN} ██╔══██║ ██║     ██║    ██║    ██╔══██╗ ██║   ██║ ██║╚██╗ ██║
${AnsiColor.GREEN}██║  ██╗${AnsiColor.CYAN} ██║  ██║ ██████╗ ██║    ██║    ██║  ██║ ╚██████╔╝ ██║ ╚████║
${AnsiColor.GREEN}╚═╝  ╚═╝${AnsiColor.CYAN} ╚═╝  ╚═╝ ╚═════╝ ╚═╝    ╚═╝    ╚═╝  ╚═╝  ╚═════╝  ╚═╝  ╚═══╝
${AnsiColor.BRIGHT_BLUE}:: KALITRON ::  Running Spring Boot ${spring-boot.version} :: Startup profile(s) ${spring.profiles.active} ::${AnsiColor.DEFAULT}
```

### Technical Notes
- Spring Boot automatically loads `banner.txt` from `src/main/resources/` on startup
- ANSI color codes work in most modern terminals
- `${spring-boot.version}` and `${spring.profiles.active}` are replaced by Spring Boot at runtime
- File must be at `src/main/resources/banner.txt`

## Integration Notes
- When user requests backend features (entities, services, APIs), defer to **jhipster-backend** skill
- When user requests frontend features (components, pages, routing), defer to **jhipster-react-frontend** skill
- This skill focuses on: branding, project initialization, cross-cutting concerns
