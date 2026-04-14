# Requirements: Videoclub STARTUP — Series Tracking

**Defined:** 2026-04-14
**Core Value:** Saber exactamente en qué punto dejaste una serie y poder marcar episodios vistos de forma rápida sin salir de la app.

## v1 Requirements

### Base de datos

- [ ] **DB-01**: La tabla `items` incluye columna `tmdb_id TEXT` (migración no destructiva, idempotente)
- [ ] **DB-02**: La tabla `items` incluye columna `num_seasons INTEGER` para evitar llamadas extra a la API
- [ ] **DB-03**: Nueva tabla `episode_progress (user_id, tmdb_series_id, season_number, episode_number, watched, watched_at)` con PK compuesta
- [ ] **DB-04**: `SCHEMA_VERSION` bumpeado a 7; migración ejecuta solo si `v < 7`
- [ ] **DB-05**: `tmdb_id` se persiste al añadir o enriquecer un ítem de tipo serie vía TMDb

### API TMDb — Temporadas

- [ ] **API-01**: Función `fetch_tmdb_seasons(tmdb_id)` obtiene la lista de temporadas desde `/tv/{id}` (sin llamada extra — datos ya en detail response)
- [ ] **API-02**: Función `fetch_tmdb_season_episodes(tmdb_id, season_number)` obtiene episodios desde `/tv/{id}/season/{n}`
- [ ] **API-03**: La lista de temporadas se construye desde el array `seasons[]` del detail response (no `range(1, N+1)`) para capturar Season 0 y numeración irregular
- [ ] **API-04**: Caché `@lru_cache(maxsize=50)` en `fetch_tmdb_season_episodes` para evitar refetching dentro de la misma sesión
- [ ] **API-05**: Error handling visible al usuario cuando TMDb no devuelve datos de una temporada

### Vista de Temporadas (SeasonViewWindow)

- [ ] **UI-01**: Botón "Ver temporadas" en `DetailWindow` visible solo si el ítem es tipo `series` y tiene `tmdb_id`
- [ ] **UI-02**: `SeasonViewWindow(tk.Toplevel)` abre con su propio `grab_set()`; restaura el grab de `DetailWindow` al cerrarse
- [ ] **UI-03**: Tabs de temporadas con `ttk.Notebook` + `<<NotebookTabChanged>>`; carga lazy (solo la tab activa)
- [ ] **UI-04**: Indicador de progreso por tab: `"X / Y episodios vistos"` en el header de cada temporada
- [ ] **UI-05**: Lista de episodios con `Canvas` + `Frame` scrollable (dark theme consistente); muestra número, título y fecha de emisión por fila
- [ ] **UI-06**: Checkbox por episodio (Tkinter `Checkbutton` con `BooleanVar`); toggle escribe a SQLite inmediatamente
- [ ] **UI-07**: Botón "Marcar toda la temporada como vista" en el header de cada temporada
- [ ] **UI-08**: Episodios con `air_date` futuro aparecen en gris (no se pueden marcar como vistos)
- [ ] **UI-09**: Estado de carga visible (spinner/label) mientras se obtienen datos de TMDb; error visible si falla
- [ ] **UI-10**: Al cerrar `SeasonViewWindow`, se llama `on_refresh` para que el grid y `DetailWindow` reflejen el progreso

### Indicador de progreso en grid

- [ ] **GRID-01**: Las tarjetas de tipo `series` con episodios marcados muestran un badge de progreso (e.g. `T2 E5`) sin necesidad de abrir la ventana de detalle

### Series sin tmdb_id (retrocompatibilidad)

- [ ] **BACK-01**: Series añadidas antes de v7 con `tmdb_id = NULL` muestran el botón "Ver temporadas" deshabilitado con tooltip explicativo
- [ ] **BACK-02**: Al hacer clic en el botón deshabilitado, se sugiere re-añadir la serie vía búsqueda para activar el tracking

## v2 Requirements

### Mejoras de UX

- **UX2-01**: "Marcar vistos todos los episodios hasta el episodio N" (clic derecho o botón inline)
- **UX2-02**: Scroll automático al primer episodio no visto al abrir una temporada
- **UX2-03**: Secciones de temporada colapsables para series largas

### Enriquecimiento masivo

- **ENRICH-01**: Diálogo "Enriquecer series existentes" para backfill de `tmdb_id` en series pre-v7 (similar a `RefreshMetaDialog` existente)

### Estadísticas

- **STATS-01**: Panel de estadísticas: episodios vistos totales, tiempo estimado, series completadas

## Out of Scope

| Feature | Razón |
|---------|-------|
| Valoración por episodio | Complejidad de DB sin valor claro para v1; fuera de scope explícito en PROJECT.md |
| Diálogos de confirmación al marcar/desmarcar | Añade fricción, la acción es trivialmente reversible |
| Paginación dentro de una temporada | 6-26 eps siempre caben en una lista scrollable |
| Imágenes still por episodio | Una llamada extra de API por episodio, presión de caché, sin valor de tracking |
| Notificaciones de nuevos episodios | Requiere polling continuo, fuera del alcance de la app desktop offline-first |
| Sincronización con Trakt.tv / Letterboxd | Integración externa compleja; siguiente milestone |
| Links de streaming por episodio | Feature separado, no relacionado con el tracking |

## Traceability

| Requisito | Fase | Estado |
|-----------|------|--------|
| DB-01 | Fase 1 | Pending |
| DB-02 | Fase 1 | Pending |
| DB-03 | Fase 1 | Pending |
| DB-04 | Fase 1 | Pending |
| DB-05 | Fase 1 | Pending |
| API-01 | Fase 1 | Pending |
| API-02 | Fase 1 | Pending |
| API-03 | Fase 1 | Pending |
| API-04 | Fase 1 | Pending |
| API-05 | Fase 1 | Pending |
| UI-01 | Fase 2 | Pending |
| UI-02 | Fase 2 | Pending |
| UI-03 | Fase 2 | Pending |
| UI-04 | Fase 2 | Pending |
| UI-05 | Fase 2 | Pending |
| UI-06 | Fase 2 | Pending |
| UI-07 | Fase 2 | Pending |
| UI-08 | Fase 2 | Pending |
| UI-09 | Fase 2 | Pending |
| UI-10 | Fase 2 | Pending |
| GRID-01 | Fase 3 | Pending |
| BACK-01 | Fase 3 | Pending |
| BACK-02 | Fase 3 | Pending |

**Coverage:**
- v1 requirements: 23 total
- Mapped to phases: 23
- Unmapped: 0 ✓

---
*Requirements defined: 2026-04-14*
*Last updated: 2026-04-14 after initial definition*
