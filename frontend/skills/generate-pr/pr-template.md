## Pull Request

### Título: `<type>(<scope>): <short description>`

Usa Conventional Commits. Ejemplos: `feat(catalog): implement search filters`, `fix(auth): handle token expiration`, `refactor(rental): simplify state machine`

---

### 📝 Contexto y Motivación

- Explicación breve (2-3 oraciones): ¿Qué problema se resolvió? ¿Es un bugfix, feature, refactor o mejora?

---

### 🛠️ Detalles de Implementación

Resume los cambios clave por capa (sé conciso, agrupa por responsabilidad, no listes archivos uno por uno):

- **Presentación/UI:** Flujos/páginas principales, estados de carga/error, cambios en BLoC/Cubit
- **Dominio/Casos de Uso:** Casos de uso creados/modificados, modelos de dominio, validadores
- **Datos/Integración:** DTOs, repositorios, data sources, mapeo DTO↔Dominio
- **Core:** Routing, DI, tema, manejo de errores, widgets compartidos
- **Refactors:** Comportamiento sin cambios, objetivos (legibilidad, performance, etc.)

_(Omite secciones que no apliquen al PR)_

---

### 📦 Cambios Estructurales

- [ ] Nuevo BLoC/Cubit
- [ ] Nuevo caso de uso o repositorio
- [ ] Cambio en routing (`app_router.dart`)
- [ ] Cambio en DI (`injection.dart` / nuevas anotaciones)
- [ ] Dependencias (`pubspec.yaml`)
- [ ] Cambio en design system (`AppColors`, `AppTheme`, `AppTypography`)
- [ ] Ninguno (solo interno)

---

### 📸 Visuales (Cambios de UI)

Si no hay cambios de UI: `No hay cambios en la UI.`

Si hay cambios de UI, el developer debe agregar capturas/GIFs manualmente:
- **Antes:** Arrastra y suelta capturas aquí
- **Después:** Arrastra y suelta capturas aquí (incluye estados de carga/error)
- **Estados:** [ ] Loading  [ ] Error  [ ] Success  [ ] Empty

---

### ⚠️ Cambios disruptivos

Si no hay: `Cambios disruptivos: Ninguno.`

Si sí hay: lista los contratos/interfaces cambiados, los casos de uso o repositorios afectados, y si requiere migración de datos.
