# Videoclub STARTUP — Series Tracking

## What This Is

Videoclub STARTUP es una aplicación de escritorio (Python + Tkinter) para gestionar una biblioteca personal de películas y series. El proyecto actual (v7.0) añade seguimiento por temporada y episodio a las series: una vista dedicada que carga los datos desde TMDb y permite marcar episodios individualmente.

## Core Value

Saber exactamente en qué punto dejaste una serie y poder marcar episodios vistos de forma rápida sin salir de la app.

## Requirements

### Validated

- ✓ Biblioteca multi-usuario con items (películas/series) y estados watched/pending/none — existing
- ✓ Búsqueda multi-fuente: OMDb + TMDb + DuckDuckGo con resultados múltiples — existing
- ✓ Posters async con LRU cache (200 entradas, ThreadPool 4 workers) — existing
- ✓ Virtual scroll + paginación (36 items/página) — existing
- ✓ Importación batch desde .txt/.csv — existing
- ✓ Export a CSV/JSON — existing
- ✓ Atajos de teclado: Ctrl+N, Ctrl+F, F5, Escape — existing
- ✓ Setup wizard para API keys (OMDb, TMDb) — existing
- ✓ Motor de recomendación aleatorio filtrable por género — existing

### Active

- [ ] Vista de temporadas: ventana nueva que muestra todas las temporadas de una serie con sus episodios
- [ ] Carga de datos de temporadas/episodios desde TMDb API (endpoint `/tv/{id}/season/{n}`)
- [ ] Marcar episodios individuales como vistos con checkbox por episodio
- [ ] Progreso por temporada visible en la vista (X/Y episodios vistos)
- [ ] Persistencia del progreso en SQLite (nueva tabla `episode_progress`)
- [ ] Acceso desde la ventana de detalle de una serie (botón "Ver temporadas")
- [ ] Indicador de progreso en la tarjeta del grid para series con episodios marcados

### Out of Scope

- Tracking por episodio para películas — no aplica por naturaleza
- Notificaciones de nuevos episodios — requiere polling continuo, fuera de alcance v1
- Sincronización con Trakt.tv / Letterboxd — integración externa compleja, siguiente milestone
- Valoración por episodio — añade complejidad de DB sin valor claro para v1
- Descarga de subtítulos o streaming — fuera del propósito de la app

## Context

- **Codebase**: Fichero único `Videoclub` (~3650 líneas), sin extensión, ejecutado directamente por Python. Compilado a .exe con PyInstaller.
- **UI**: Tkinter puro + Pillow. Dark theme con paleta fija en el dict `C`. Fuente Segoe UI en Windows.
- **DB**: SQLite en `%APPDATA%/Videoclub STARTUP/videoclub.db`. Schema v6 con tablas: `items`, `users`, `watched`, `user_items`, `meta`.
- **TMDb**: Ya integrado para búsqueda (`fetch_tmdb_multi`, `fetch_tmdb_detail`). La API key se guarda en `config.json`. El endpoint de temporadas es `/tv/{series_id}/season/{season_number}`.
- **Series en DB**: El campo `type` diferencia `movie` de `series`. El `id` de TMDb no se persiste actualmente — necesario para llamar a la API de temporadas.
- **Threading**: Operaciones de red en ThreadPoolExecutor. La UI solo se toca desde el hilo principal.

## Constraints

- **Tech stack**: Python + Tkinter — no cambiar el stack base, la app se distribuye como .exe standalone
- **Compatibilidad**: Windows 10/11 (uso principal). El .exe debe seguir funcionando sin instalación de Python
- **API**: TMDb API key requerida para cargar episodios — si no hay key, mostrar aviso claro
- **DB**: Migración no destructiva — el schema v6 existente no se rompe. Añadir schema v7 con nueva tabla
- **Fichero único**: El código vive en un solo fichero `Videoclub`. No refactorizar a módulos múltiples

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| TMDb como fuente de datos de episodios | Tiene endpoint detallado por temporada con nombres, fechas y sinopsis de cada ep | — Pending |
| Nueva tabla `episode_progress` en lugar de extender `watched` | `watched` usa `title` como FK y es para el ítem completo; episodios necesitan granularidad distinta | — Pending |
| Persistir `tmdb_id` en tabla `items` | Necesario para llamar a la API de temporadas sin re-buscar | — Pending |
| Ventana nueva para vista de temporadas (Toplevel) | Consistente con DetailWindow y otros diálogos de la app | — Pending |

## Evolution

Este documento evoluciona en cada transición de fase y milestone.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-04-14 after initialization*
