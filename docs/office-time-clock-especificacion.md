# Office Time Clock — Especificación Técnica Final

**Repositorio:** `office-time-clock`
**Descripción:** A lightweight Node.js + SQLite web app for clocking in/out in a small office. Time is validated server-side against an external HTTPS source to prevent local clock manipulation.

Este documento consolida la especificación técnica original del proyecto junto con todas las decisiones de diseño tomadas durante la sesión de planificación. Es el documento de referencia único para arrancar la implementación.

---

## 1. Objetivo

Desarrollar una aplicación web local para un despacho pequeño de máximo 2 empleados, cuya función principal sea registrar entradas y salidas de manera sencilla.

El sistema debe:

- Funcionar principalmente de manera local.
- Registrar la hora de entrada y salida de cada empleado.
- No depender de la hora configurada en la PC como fuente confiable.
- Obtener/sincronizar la hora con una fuente externa de Internet.
- Guardar los registros localmente.
- Permitir consultar los registros.
- Generar un archivo de respaldo/exportación semanal.
- Notificar al jefe de cada checada.
- Ser sencillo de instalar, mantener y utilizar.
- No requerir infraestructura empresarial ni un sistema de respaldos complejo.

## 2. Alcance

Diseñado para una sola oficina/despacho con:

- Máximo 2 empleados (Juan y María).
- Una PC Windows como dispositivo principal.
- Un administrador/jefe.
- Pocos registros diarios.

**No se requiere inicialmente:** multiempresa, múltiples sucursales, nómina, servidores en la nube, alta disponibilidad, backups empresariales automatizados, roles y permisos complejos, escalabilidad para cientos de usuarios.

La prioridad es simplicidad, confiabilidad y bajo mantenimiento — evitar sobreingeniería en todo momento.

## 3. Arquitectura

```
                ┌───────────────────────┐
                │      INTERNET         │
                │  Fuente de tiempo:    │
                │  header Date (HTTPS)  │
                └───────────┬───────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────┐
│                 PC DEL DESPACHO (Windows)        │
│                                                   │
│  ┌───────────────────────────────────────────┐  │
│  │      Frontend HTML/CSS/JavaScript          │  │
│  └─────────────────────┬─────────────────────┘  │
│                        │                         │
│                        ▼                         │
│  ┌───────────────────────────────────────────┐  │
│  │      Backend Node.js (Express/Fastify)     │  │
│  │  API + lógica de negocio + hora confiable  │  │
│  │  gestionado por pm2                        │  │
│  └───────────────┬───────────────┬───────────┘  │
│                  ▼               ▼               │
│           ┌────────────┐   ┌──────────────┐     │
│           │   SQLite   │   │ emailService │     │
│           │  local.db  │   │  (correo)    │     │
│           └────────────┘   └──────────────┘     │
└─────────────────────────────────────────────────┘
                         │
                         │ Backup semanal manual
                         ▼
                  ┌──────────────────┐
                  │ TXT / CSV +       │
                  │ database.sqlite  │
                  └──────┬───────────┘
                         │
                         ▼
                    Google Drive (subida manual)
```

## 4. Stack tecnológico

| Componente | Elección | Motivo |
|---|---|---|
| Backend | Node.js + Express/Fastify | Simplicidad, ecosistema amplio |
| Base de datos | SQLite (`better-sqlite3`) | Proyecto pequeño y local, no requiere servidor de BD |
| Frontend | HTML/CSS/JS vanilla | Evita complejidad innecesaria; React opcional si aporta ventaja real |
| Gestión de proceso | pm2 | Reinicio automático ante fallos y arranque con Windows |
| Notificaciones | Email (SMTP o proveedor tipo Resend) | Reemplaza WhatsApp para el MVP; gratis, sin trámites |
| Fuente de hora | Header HTTP `Date` vía HTTPS (ej. Google) | Evita depender de APIs de tiempo de terceros que se descontinúan |

### API REST local

```
GET  /api/empleados
POST /api/registro
GET  /api/registros
GET  /api/registros/semana
GET  /api/exportar
GET  /api/estado
```

## 5. Funcionamiento principal

Pantalla principal simple, un bloque por empleado con botones grandes:

```
┌────────────────────┐      ┌────────────────────┐
│       JUAN         │      │      MARÍA          │
│     [ ENTRADA ]    │      │     [ ENTRADA ]    │
│     [  SALIDA ]    │      │     [  SALIDA ]    │
└────────────────────┘      └────────────────────┘
```

## 6. Flujo de entrada

El frontend **nunca** envía la hora. Solo:

```json
{
  "empleadoId": 1,
  "tipo": "entrada"
}
```

El backend determina timestamp, fecha, hora, zona horaria, fuente de tiempo y validación de jornada.

## 7. Fuente confiable de hora — DECISIÓN FINAL

**No se usa NTP ni una API dedicada de tiempo** (ej. worldtimeapi.org — descontinuada, o sus reemplazos comunitarios como timeapi.world, que son proyectos de terceros con riesgo de caída/descontinuación).

**Se usa:** el header HTTP `Date` (siempre en UTC/GMT por especificación del protocolo HTTP) de una petición `HEAD` a un sitio grande y estable (ej. `https://www.google.com`). Ventajas:

- No requiere API key ni cuenta.
- Google/Cloudflare prácticamente nunca están caídos.
- Se pueden usar varios sitios como fallback (google.com, cloudflare.com, microsoft.com).

```javascript
const https = require('https');

function obtenerHoraExterna() {
  return new Promise((resolve, reject) => {
    const req = https.request(
      { hostname: 'www.google.com', method: 'HEAD' },
      (res) => resolve(new Date(res.headers['date']))
    );
    req.on('error', reject);
    req.end();
  });
}
```

## 8. Estrategia de sincronización

```
Servidor inicia → Obtiene hora externa (header Date) → Compara contra reloj local
     → Calcula offset → Guarda offset
```

`hora_confiable = hora_local + offset`. Resincronización periódica (ej. cada 30 min, configurable vía `.env`).

## 9. Funcionamiento offline

Si se pierde Internet, el sistema sigue funcionando con la última sincronización conocida, marcando internamente `source: cached_time`. Al volver Internet, se resincroniza y actualiza el offset.

Si el sistema nunca ha logrado sincronizar desde la instalación, debe advertir y preferiblemente impedir registrar una checada.

## 10. Evitar manipulación de la hora

El frontend nunca es responsable de determinar la hora. No usar `new Date()` del navegador como fuente de verdad. Toda la lógica de tiempo vive en el backend.

## 11. Base de datos

### Tabla `empleados`

```sql
CREATE TABLE empleados (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nombre TEXT NOT NULL,
    activo INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL
);
```

**Baja de empleados = soft delete.** Nunca se borra la fila; se marca `activo = 0`. El empleado desactivado deja de aparecer en la pantalla del checador (`GET /api/empleados` filtra `activo = 1`), pero su historial se conserva íntegro.

### Tabla `registros`

```sql
CREATE TABLE registros (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    empleado_id INTEGER NOT NULL,
    tipo TEXT NOT NULL,              -- 'entrada' | 'salida'
    timestamp_utc TEXT NOT NULL,
    timezone TEXT NOT NULL,
    source TEXT NOT NULL,            -- 'internet' | 'cached_time'
    flag TEXT DEFAULT NULL,          -- ver sección 12 (olvido de salida)
    created_at TEXT NOT NULL,
    FOREIGN KEY (empleado_id) REFERENCES empleados(id)
);
```

### Tabla `auditoria`

```sql
CREATE TABLE auditoria (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    registro_id INTEGER,
    accion TEXT NOT NULL,
    valor_anterior TEXT,
    valor_nuevo TEXT,
    fecha_utc TEXT NOT NULL
);
```

**Incluida desde el MVP** (no es opcional): toda edición o eliminación de un registro desde el panel de administración debe pasar por esta tabla, registrando qué cambió y cuándo.

### Tabla `admin`

Almacena el PIN de acceso al panel de administración.

- **DECISIÓN:** el PIN se guarda en **texto plano** (sin hash/encriptación), ya que es una app local con usuarios no técnicos y el riesgo real ya depende de quién tenga acceso físico a la PC.
- Si se olvida, se consulta o edita directamente en la base de datos — no hay flujo de recuperación.

## 12. Reglas de negocio

### Secuencia válida

```
entrada → salida → entrada → salida
```

No se permite `entrada → entrada` consecutiva (mientras la entrada abierta sea del **mismo día**).

### Olvido de checar salida — DECISIÓN FINAL

Si un empleado tiene una entrada abierta (sin salida):

- **Si la entrada abierta es del día actual:** se sigue bloqueando una nueva entrada, con el mensaje de error habitual.
- **Si la entrada abierta es de un día anterior:** se **permite** la nueva entrada de hoy, y el registro viejo (el del día anterior) se marca con la columna `flag = 'sin_salida_olvidada'`.
- En el panel de administración, los registros con este flag se resaltan visualmente (ej. ⚠️) para que el admin los note al revisar el historial.
- Cuando el admin corrige manualmente ese registro (le agrega la hora de salida), el flag **no desaparece** — cambia de valor a `flag = 'corregido_por_admin'`, para mantener rastro de que hubo un olvido y que fue resuelto. Esta corrección queda además registrada en la tabla `auditoria`.

### Salida sin entrada abierta

No debe permitirse `salida` si no existe una `entrada` abierta. Mensaje: `❌ No existe una entrada pendiente para este empleado.`

## 13. Evitar doble clic

El botón se deshabilita temporalmente tras presionarse (frontend). El backend también valida esto de forma independiente — no confiar únicamente en el frontend.

## 14. Confirmación visual

Tras registrar, mostrar confirmación (empleado, tipo, hora) y volver automáticamente a la pantalla principal tras unos segundos.

## 15. Historial

Pantalla administrativa con filtros mínimos: fecha, empleado, semana. Los registros con `flag` activo se resaltan.

## 16. Resumen diario (opcional)

Horas trabajadas por empleado; si no hay salida, mostrar `Estado: Jornada abierta`.

## 17. Exportación y backup — DECISIÓN FINAL

Botón `[EXPORTAR SEMANA]` / `[EXPORTAR BACKUP]` genera:

```
backup-2026-08-29/
├── checador-2026-08-24-a-2026-08-30.txt
├── checador-2026-08-24-a-2026-08-30.csv
└── database.sqlite   ← copia completa de la base de datos
```

- **Se incluye una copia completa del archivo `database.sqlite`**, no solo TXT/CSV — así, ante una falla de disco, se puede restaurar el sistema completo (empleados, PIN, flags, auditoría) reemplazando el `.sqlite` en una PC nueva.
- **La subida a Google Drive se mantiene manual** (se decidió explícitamente no automatizar esto en el MVP, para no añadir la complejidad de integrar la API de Google Drive — OAuth, tokens, manejo de errores).

## 18. Notificación al jefe — DECISIÓN FINAL (reemplaza WhatsApp)

**Canal elegido para el MVP: correo electrónico**, enviado automáticamente desde el backend tras cada checada — sin intervención humana.

Se evaluaron y descartaron para el MVP:

- **WhatsApp Business Cloud API oficial:** técnicamente cumple el requisito de envío 100% automático, pero requiere cuenta de Meta Business verificada, número dedicado, aprobación de plantillas de mensaje, y tiene costo por mensaje (aunque mínimo dado el volumen). Se descarta por la fricción de configuración inicial, no por limitación técnica — queda como posible mejora futura.
- **Enlace `wa.me`:** gratis y simple, pero requiere que un humano presione "Enviar" manualmente — no cumple el requisito de automatización real.
- **Telegram (bot):** automático, gratuito, sin trámites — alternativa viable si en el futuro se prefiere una notificación tipo chat en vez de correo.
- **SMS (Twilio):** automático, con costo por mensaje similar a WhatsApp; descartado por simplicidad frente a email.

Patrón de implementación (se mantiene desacoplado, igual que en el diseño original):

```javascript
emailService.sendAttendanceNotification(...)
```

Esto permite cambiar de canal en el futuro (a Telegram, WhatsApp, SMS) sin modificar la lógica principal del checador.

## 19. Seguridad

**Backend:**
- Validar todos los parámetros; no confiar en datos del navegador.
- No aceptar una hora enviada por el frontend.
- Validar `empleadoId` y `tipo`.
- Queries parametrizadas.
- No exponer la base SQLite directamente.
- Acceso administrativo protegido con PIN (ver sección 11).

**Frontend:** únicamente envía `{ empleadoId, tipo }`. Toda la lógica importante vive en el backend.

## 20. Administrador

Sección `/administracion` protegida con PIN. Permite:

- Ver historial (con registros marcados por `flag`).
- Exportar semana / backup (incluyendo `.sqlite`).
- Administrar empleados (alta y baja vía soft delete).
- Ver estado de sincronización de hora.
- **Editar/eliminar registros manualmente**, con auditoría completa (tabla `auditoria`).

## 21. Estado de sincronización

El panel de administración debe mostrar:

```
Hora del sistema:        15:03:27
Hora sincronizada:       15:03:26
Última sincronización:   15:00:00
Estado:                  ✓ Sincronizado / ⚠ Sin Internet
Modo:                    Hora basada en última sincronización (si aplica)
```

## 22. Estructura del proyecto

```
office-time-clock/
│
├── package.json
├── server.js
├── .env
├── .gitignore
│
├── database/
│   ├── database.sqlite
│   └── migrations/
│
├── src/
│   ├── routes/
│   │   ├── empleados.js
│   │   ├── registros.js
│   │   └── admin.js
│   │
│   ├── services/
│   │   ├── timeService.js         (header Date via HTTPS)
│   │   ├── attendanceService.js
│   │   ├── emailService.js        (reemplaza whatsappService)
│   │   └── exportService.js       (incluye copia de .sqlite)
│   │
│   └── database/
│       └── db.js
│
└── public/
    ├── index.html
    ├── admin.html
    ├── css/
    │   └── styles.css
    └── js/
        ├── app.js
        └── admin.js
```

## 23. Variables de entorno

```
PORT=3000
TIME_SYNC_INTERVAL=1800000
TIMEZONE=America/Mexico_City
TIME_SOURCE_HOSTS=www.google.com,www.cloudflare.com

EMAIL_ENABLED=true
EMAIL_PROVIDER=smtp        # o resend
JEFE_EMAIL=
SMTP_HOST=
SMTP_USER=
SMTP_PASS=
```

## 24. Instalación

```
npm install
npm start
```

Abrir `http://localhost:3000`.

**Arranque automático (DECISIÓN FINAL):** se usa **pm2**, configurado una sola vez durante la instalación:

```
npm install -g pm2
pm2 start server.js --name checador
pm2 save
pm2 startup
```

Con esto, el proceso se reinicia solo ante fallos y arranca automáticamente al reiniciar Windows — sin volver a correr `npm start` manualmente.

## 25. Requisitos funcionales del MVP

- [ ] Pantalla principal del checador (2 empleados).
- [ ] Botones de entrada/salida.
- [ ] Registro en SQLite.
- [ ] Timestamp generado por backend, vía header `Date` (HTTPS).
- [ ] Re-sincronización periódica + modo offline con `source: cached_time`.
- [ ] Validación entrada/salida, incluyendo manejo de olvido de salida (`flag`).
- [ ] Protección contra doble registro.
- [ ] Historial con filtros.
- [ ] Exportación semanal (TXT/CSV + copia de `.sqlite`).
- [ ] Notificación automática por correo electrónico.
- [ ] Panel administrativo con PIN (texto plano), edición/eliminación con auditoría.
- [ ] Estado de sincronización visible.
- [ ] Arranque automático vía pm2.
- [ ] Baja de empleados vía soft delete.

## 26. Requisitos no funcionales

- **Simplicidad:** priorizar siempre sobre robustez innecesaria para esta escala.
- **Confiabilidad:** un fallo de Internet no debe provocar pérdida de registros.
- **Seguridad:** el usuario no debe poder falsificar la hora desde el navegador.
- **Usabilidad:** registrar una checada requiere máximo 2 pasos (identificar empleado, presionar botón).
- **Mantenimiento:** el backup semanal no debe requerir conocimientos avanzados.

## 27. Filosofía general

No sobreingenierizar. El sistema es para un despacho de máximo dos personas. Priorizar:

```
Simplicidad + Confiabilidad + Bajo mantenimiento + Hora difícil de manipular + Datos locales
```

sobre microservicios, Kubernetes, nube compleja, bases de datos distribuidas, CI/CD empresarial o alta disponibilidad.

## 28. Mejoras futuras (fuera del MVP)

- Reportes mensuales, cálculo automático de horas, retardos/tolerancias, días festivos, vacaciones, incapacidades.
- Exportación a Excel.
- Registro mediante QR.
- Notificación por Telegram o WhatsApp Business API como alternativa/complemento al correo.
- Automatización de la subida de backups a Google Drive.
- Dashboard, acceso remoto, base de datos centralizada, backups automáticos, app móvil/PWA.

## 29. Entorno de desarrollo (Fase 0 — completada)

- PC Windows personal.
- Node.js v24.20.0 instalado.
- Git 2.55.0 instalado.
- VSCode instalado, con sesión de GitHub iniciada.
- Repositorio `office-time-clock` (privado) creado y clonado en `C:\Development\GitHub\office-time-clock`.
