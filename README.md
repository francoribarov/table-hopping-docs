# Table Hopping: documentación de agentes

Este repositorio reúne la documentación de desarrollo asistido utilizada en
los repositorios de Table Hopping. Los contenidos están separados por
aplicación para facilitar su consulta desde la tesis.

## Estructura

- `backend/`
  - `CLAUDE.md`
  - `agents/`
  - `skills/`
  - `commands/`
- `frontend/`
  - `CLAUDE.md`
  - `agents/`
  - `skills/`

Las copias de agentes y skills que eran idénticas entre `.claude`, `.cursor` y
`.agents` se consolidan en un único archivo. Se conserva el contenido de la
versión vigente de cada documentación.

## Alcance

Se incluyen las guías `CLAUDE.md`, las definiciones de agentes, las skills y
los archivos auxiliares de cada skill. Se excluyen deliberadamente
`AGENTS.md`, `GEMINI.md`, configuraciones locales y borradores de pull
requests.
