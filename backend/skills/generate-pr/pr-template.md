## Pull Request

### Titulo: `<type>(<scope>): <short description>`

Usa Conventional Commits. Ejemplos: `feat(rentals): add drop-off response flow`,
`fix(auth): validate refresh token issuer`, `refactor(users): simplify service wiring`

---

### Contexto y Motivacion

- Explicacion breve (2-3 oraciones): Que problema se resolvio? Es bugfix, feature,
  refactor o mejora?

---

### Detalles de Implementacion

Resume los cambios clave por capa (se conciso, agrupa por responsabilidad, no
listes archivos uno por uno):

- **API:** Endpoints, schemas, dependencias, codigos HTTP, validaciones
- **Dominio:** Entidades, interfaces, servicios, reglas de negocio, excepciones
- **Infraestructura:** Repositorios SQLAlchemy, mappers, adapters externos
- **Core:** Configuracion, seguridad, middleware, manejo de errores
- **Base de datos:** Migraciones Alembic, cambios de esquema
- **Tests:** Unit/integration, fixtures, cobertura en caminos criticos
- **Refactors:** Comportamiento sin cambios, objetivos (legibilidad, mantenibilidad)

_(Omite secciones que no apliquen al PR)_

---

### Cambios Estructurales

- [ ] Nuevo endpoint o ruta en `app/api/endpoints/`
- [ ] Nuevo schema Pydantic (`*Schema`)
- [ ] Nuevo servicio de dominio o interfaz
- [ ] Nuevo repositorio o mapper en infraestructura
- [ ] Cambio en inyeccion de dependencias (`service_dependencies.py` / `repository_dependencies.py`)
- [ ] Cambio en seguridad/autenticacion
- [ ] Migracion Alembic
- [ ] Cambio en contratos de respuesta/API publica
- [ ] Ninguno (solo interno)

---

### Riesgos y Compatibilidad

Si no hay: `Riesgos: bajos. Compatibilidad: sin cambios disruptivos.`

Si si hay: detalla impacto en contratos API, migraciones requeridas y pasos de
rollback.

---

### Cambios disruptivos

Si no hay: `Cambios disruptivos: Ninguno.`

Si si hay: lista endpoints o contratos alterados, payloads afectados, y estrategia
de migracion para clientes consumidores.
