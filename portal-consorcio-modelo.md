# Portal del Consorcio — Modelo de Datos

## Concepto

El portal del consorcio es una vista de solo lectura accesible mediante un **link con token único por edificio** (sin cuenta, sin contraseña). Permite a la administradora o encargado del edificio ver el historial de mantenimiento de sus equipos, recibir confirmaciones post-visita y reportar fallas.

URL de ejemplo:
```
liftmanager.app/edificio/{token}
```

El token es opaco (UUID v4 o similar), sin información codificada. La empresa de ascensores genera y comparte este link una sola vez; el consorcio lo bookmarkea.

---

## Entidades principales

### `Building` (Edificio)

```
id                UUID, PK
tenant_id         UUID, FK → Tenant (empresa de ascensores)
name              string         — "Torres del Parque, Av. Libertador 2500"
address           string
contact_name      string         — nombre del admin del consorcio
contact_email     string         — destino de notificaciones automáticas
contact_phone     string?
portal_token      UUID, unique   — token público del portal, generado al crear
portal_token_active  boolean, default true
created_at        timestamp
updated_at        timestamp
```

**Notas:**
- `portal_token` es inmutable salvo regeneración explícita (opción "revocar link" en el admin).
- `contact_email` es el campo que dispara las notificaciones post-visita.

---

### `Equipment` (Equipo)

```
id                UUID, PK
building_id       UUID, FK → Building
tenant_id         UUID, FK → Tenant
type              enum: 'ascensor' | 'montauto' | 'porton' | 'rampa' | 'otro'
label             string         — "Ascensor A", "Porton planta baja"
floor             string?        — "PB", "Piso 3-8", etc.
brand             string?
model             string?
serial_number     string?
installation_year integer?
qr_code           string, unique — código del QR físico adherido al equipo
next_maintenance_date  date?
is_active         boolean, default true
created_at        timestamp
```

**Notas:**
- `qr_code` apunta a `liftmanager.app/qr/{qr_code}`. La misma URL sirve para el técnico (autenticado) y para el público (sin sesión). La lógica de renderizado difiere según autenticación.
- El QR físico no cambia; `qr_code` es estable una vez asignado.

---

### `WorkOrder` (Orden de Trabajo)

```
id                UUID, PK
tenant_id         UUID, FK → Tenant
building_id       UUID, FK → Building
equipment_id      UUID, FK → Equipment
technician_id     UUID, FK → User (técnico)
type              enum: 'preventivo' | 'correctivo' | 'emergencia' | 'instalacion'
status            enum: 'pendiente' | 'en_curso' | 'completada' | 'cancelada'
description       text?          — descripción del trabajo a realizar
technician_notes  text?          — observaciones al cerrar
photos            string[]       — URLs de fotos adjuntas al cierre
parts_used        jsonb?         — [{ name, quantity, cost }]
scheduled_date    date?
opened_at         timestamp
started_at        timestamp?
closed_at         timestamp?
created_at        timestamp
```

**Campos visibles en el portal (solo lectura):**
- `type`, `status`, `description`, `technician_notes`, `photos`, `scheduled_date`, `closed_at`
- Nombre del técnico (no su ID ni datos de contacto)

**Campos ocultos al portal:**
- `parts_used` (costos internos), `technician_id` directo

---

### `FaultReport` (Reporte de Falla)

Generado desde el QR público por vecinos sin autenticación.

```
id                UUID, PK
equipment_id      UUID, FK → Equipment
building_id       UUID, FK → Building
tenant_id         UUID, FK → Tenant
reporter_name     string?        — opcional, el vecino puede omitirlo
reporter_contact  string?        — email o teléfono, opcional
description       text
photos            string[]       — URLs de fotos adjuntas
urgency           enum: 'normal' | 'urgente'
status            enum: 'nuevo' | 'en_revision' | 'resuelto'
work_order_id     UUID?, FK → WorkOrder  — se completa cuando se crea la OT derivada
source            enum: 'qr_publico' | 'portal_consorcio' | 'admin'
ip_address        string?        — para rate limiting básico
created_at        timestamp
```

**Notas:**
- Un `FaultReport` puede generar automáticamente una `WorkOrder` (status `pendiente`) con `type = 'correctivo'`.
- La empresa de ascensores ve todos los reportes en su panel y decide si crear una OT o descartar.

---

### `PortalNotification` (Notificación al Consorcio)

Registro de cada notificación enviada al consorcio para auditoría y reenvío.

```
id                UUID, PK
building_id       UUID, FK → Building
work_order_id     UUID, FK → WorkOrder
type              enum: 'visita_completada' | 'mantenimiento_programado' | 'fault_received'
recipient_email   string         — snapshot del contact_email al momento del envío
subject           string
body_html         text
sent_at           timestamp?
status            enum: 'pendiente' | 'enviado' | 'fallido'
error_message     string?
created_at        timestamp
```

**Notas:**
- Se genera automáticamente al cerrar una `WorkOrder` (status → `completada`).
- El envío es asíncrono (cola de jobs); este registro permite reintentos y auditoría.

---

## Lo que el portal del consorcio puede ver

```
GET /api/portal/{portal_token}
```

Respuesta (sin autenticación, solo token válido):

```json
{
  "building": {
    "name": "Torres del Parque",
    "address": "Av. Libertador 2500",
    "contact_name": "Admin Torres"
  },
  "equipments": [
    {
      "id": "...",
      "label": "Ascensor A",
      "type": "ascensor",
      "floor": "PB → Piso 12",
      "next_maintenance_date": "2026-05-10",
      "last_work_order": {
        "type": "preventivo",
        "technician_name": "Juan Ramírez",
        "closed_at": "2026-02-28T14:32:00Z",
        "notes": "Se lubricaron guías y revisaron puertas. Sin novedades.",
        "photos": ["https://..."]
      }
    }
  ],
  "recent_work_orders": [ ... ]  // últimas 10, todos los equipos
}
```

**Reglas de acceso del portal:**
- Solo muestra órdenes con `status = 'completada'`.
- No expone costos, repuestos, ni datos de facturación.
- Si `portal_token_active = false`, retorna 404 (link revocado).
- Rate limiting: máx 60 requests/hora por IP.

---

## Flujo de notificación post-visita

```
WorkOrder.status → 'completada'
        ↓
trigger: create PortalNotification (type = 'visita_completada')
        ↓
job queue: render email template con datos de la OT
        ↓
send email → building.contact_email
        ↓
PortalNotification.status → 'enviado' | 'fallido'
```

El email incluye:
- Nombre del técnico, tipo de trabajo, fecha
- Observaciones del cierre
- Fotos (thumbnails con link)
- Próximo mantenimiento programado (si existe)
- Link al portal: `liftmanager.app/edificio/{portal_token}`

---

## Flujo de reporte de falla por QR público

```
vecino escanea QR → liftmanager.app/qr/{qr_code}
        ↓
sistema detecta: sin sesión activa → vista pública del equipo
        ↓
vecino completa form: descripción + foto + urgencia (opcional: nombre/contacto)
        ↓
create FaultReport
        ↓
notificación interna a la empresa (push / email interno)
        ↓
la empresa crea WorkOrder desde el panel (o auto-creación si urgente)
```

---

## Índices recomendados

```sql
-- Búsqueda por token (frecuente, debe ser O(1))
CREATE UNIQUE INDEX idx_buildings_portal_token ON buildings(portal_token);

-- Búsqueda por QR de equipo (frecuente)
CREATE UNIQUE INDEX idx_equipments_qr_code ON equipments(qr_code);

-- OTs por edificio + estado (para el portal)
CREATE INDEX idx_work_orders_building_status ON work_orders(building_id, status);

-- Reportes de falla pendientes por empresa
CREATE INDEX idx_fault_reports_tenant_status ON fault_reports(tenant_id, status);

-- Notificaciones pendientes de envío
CREATE INDEX idx_portal_notifications_status ON portal_notifications(status) WHERE status = 'pendiente';
```

---

## Consideraciones de seguridad

- El `portal_token` no debe exponerse en URLs de la admin (IDOR). Usar siempre el `id` del edificio en la interfaz interna.
- Regenerar el token invalida el link anterior. Útil si cambia la administradora del consorcio.
- Las fotos de OTs deben servirse desde URLs firmadas con expiración (S3 presigned URLs o equivalente), no desde URLs públicas permanentes.
- Rate limiting por IP en todos los endpoints públicos (`/qr/*`, `/edificio/*`, `/api/fault-reports`).
- No exponer datos de técnicos (apellido completo, teléfono, ID) en el portal público.
