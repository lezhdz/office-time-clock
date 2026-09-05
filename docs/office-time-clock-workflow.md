# Office Time Clock — Workflow de Implementación

Este documento define cómo se va a ejecutar la implementación del proyecto, usando tres roles distintos a lo largo del proceso. El objetivo es mantener cada fase acotada, revisable, y con contexto real (no genérico) para la siguiente.

---

## 1. Roles

### A. Sesión de Planificación 
Ya cumplió su función. Produjo:
- `office-time-clock-especificacion.md` — especificación técnica completa del proyecto, con todas las decisiones de diseño ya cerradas.
- Este documento de workflow.

No vuelve a usarse. No participa en el ciclo de fases ni en la revisión.

### B. Sesión Orquestadora y Revisora ("Orch")
Una sesión **separada y persistente** durante todo el proyecto. Cumple dos funciones:

1. **Generar el prompt de cada fase:** con la especificación completa (`office-time-clock-especificacion.md`) y el estado real del código (no una idea genérica de "cómo se vería un checador", sino el código que efectivamente existe tras la fase anterior), produce el prompt de arranque de la siguiente fase de implementación — concreto, acotado, con el contexto exacto que esa fase necesita (qué ya existe, qué debe construirse, qué decisiones de la especificación aplican, y qué NO debe tocar todavía).
2. **Revisar el resultado de cada fase:** una vez la sesión de implementación produce el código, el Orch lo revisa, prueba/lee lo necesario, pide cambios o ajustes (puede devolverlos a la misma sesión de implementación mientras siga abierta), y decide cuándo la fase queda aprobada.

**El Orch nunca escribe código de producción él mismo.** Su trabajo es dirigir y validar; la escritura de código ocurre en las sesiones de implementación.

### C. Sesión de Implementación (una por fase)
Una sesión nueva **por cada fase**. Recibe el prompt generado por el Orch, escribe el código de esa fase, incorpora los ajustes que el Orch pida durante la revisión, y se cierra una vez la fase es aprobada por el Orch.

---

## 2. Flujo completo, fase por fase

```
┌─────────────────────────────────────────────────────────────┐
│  1. ORCH genera el prompt de la Fase N                       │
│     (usando la especificación + estado real del código        │
│      tras la Fase N-1)                                        │
└───────────────────────────┬────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  2. Se abre una SESIÓN DE IMPLEMENTACIÓN nueva                │
│     Se pega el prompt del Orch                                │
│     Esa sesión escribe el código de la Fase N                 │
└───────────────────────────┬────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  3. REVISIÓN (el propio ORCH)                                 │
│     El Orch revisa el código/resultado de la Fase N           │
│     ¿Necesita cambios?                                        │
│       → Sí: pide ajustes a la sesión de implementación         │
│         (mientras siga abierta)                                │
│       → No: fase aprobada                                     │
└───────────────────────────┬────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  4. Se CIERRA la sesión de implementación de la Fase N        │
│     El ORCH actualiza su propio contexto con lo realmente     │
│     implementado (archivos creados, decisiones tomadas sobre  │
│     la marcha, desviaciones del plan original, pendientes)    │
└───────────────────────────┬────────────────────────────────┘
                            ▼
                  Vuelve al paso 1, para la Fase N+1
```

**Regla clave:** el prompt de la Fase N+1 se genera **después** de cerrar y aprobar la Fase N — nunca antes. Esto es intencional: así el Orch arma el prompt con contexto real (lo que efectivamente existe en el repo) en vez de una suposición de cómo debería verse. Como el Orch es la misma sesión que revisó la fase anterior, no depende de que se le "resuma" nada desde afuera — ya tiene el contexto de primera mano.

---

## 3. Qué debe contener cada prompt del Orch

Para que cada sesión de implementación sea autosuficiente sin tener que leer todo el proyecto desde cero, cada prompt generado por el Orch debe incluir:

1. **Contexto mínimo del proyecto:** una o dos líneas de qué es Office Time Clock.
2. **Estado actual real:** qué archivos/funcionalidad ya existen (resultado de fases anteriores).
3. **Alcance exacto de esta fase:** qué se debe construir, citando las secciones relevantes de la especificación.
4. **Qué NO tocar:** para evitar que la sesión de implementación se adelante a decisiones de fases futuras o reescriba cosas ya aprobadas.
5. **Criterio de "fase terminada":** una lista corta y verificable de lo que debe funcionar al cerrar esta fase.
6. **Recordatorio de decisiones de diseño críticas** que apliquen a esa fase específica (ej. en la Fase 3, recordar explícitamente que la hora se obtiene del header `Date` vía HTTPS, no de una API dedicada).

---

## 4. Fases del proyecto (referencia)

Basado en la especificación técnica, sección 34 (ajustada con las decisiones finales):

| Fase | Contenido |
|---|---|
| 0 | Entorno de desarrollo — **completada** |
| 1 | Base: proyecto Node.js, `package.json`, Express, esquema SQLite (`empleados`, `registros`, `auditoria`, `admin`), API base |
| 2 | Interfaz del checador: pantalla principal, botones Entrada/Salida, validaciones, confirmaciones visuales |
| 3 | Hora confiable: `timeService` (header `Date` vía HTTPS), cálculo de offset, resincronización periódica, modo offline (`source: cached_time`) |
| 4 | Administración: historial con filtros y flags, exportación TXT/CSV + backup de `.sqlite`, panel admin con PIN, edición/eliminación con auditoría, gestión de empleados (soft delete), estado de sincronización |
| 5 | Notificaciones: `emailService`, envío automático al jefe tras cada checada |
| 6 | pm2: configuración de arranque automático y reinicio ante fallos |

Cada fase corresponde a una sesión de implementación distinta, en el orden anterior salvo que la revisión decida reordenar algo.

---

## 5. Documentos de referencia

- `office-time-clock-especificacion.md` — fuente única de verdad sobre requisitos y decisiones de diseño. El Orch y cada sesión de implementación deben trabajar contra este documento, no contra la memoria/interpretación de nadie.
- Este documento (`office-time-clock-workflow.md`) — define el proceso, no el contenido técnico.

Ambos documentos viven en el repositorio (ej. carpeta `/docs`) para que cualquier sesión nueva pueda leerlos directamente en vez de depender de que se le "explique" el proyecto desde cero.
