# SWNT-Proyecto-Integrador-Implementación-de-Infraestructura-con-Supabase-e-IA

## Nicolás Toro Criollo

---

Scaffolding de aplicación de gestión de inventario sostenible utilizando React(vite) y Supabase. Se aplica el Desarrollo Basado en Especificaciones (BDD) para crear el scaffolding y garantizar que la implementación cumpla con los requisitos definidos en la fase de diseño. 

## Entregables

a. Captura de pantalla/Texto del Prompt Maestro

## Bitácora de prompts

**Prompt:**

Actúa como un arquitecto de bases de datos experto en PostgreSQL.

Estoy diseñando un sistema llamado "MentorLink", una plataforma que conecta mentores con estudiantes.

Necesito que generes un script SQL profesional que incluya:

1. Tres tablas:
   - perfiles (id, nombre, bio, rol)
   - sesiones (id, mentor_id, estudiante_id, fecha, estado)
   - mensajes (id, sesion_id, remitente_id, contenido)

2. Requisitos técnicos:
   - Usa UUID como clave primaria
   - Incluye claves foráneas correctamente definidas
   - Define restricciones (NOT NULL, CHECK si aplica)
   - Usa timestamps (created_at)
   - Define relaciones coherentes entre tablas
   - Asegura integridad referencial

3. Contexto:
   - Un perfil puede ser mentor o estudiante
   - Una sesión conecta un mentor con un estudiante
   - Los mensajes pertenecen a una sesión

4. Buenas prácticas:
   - Nombres consistentes
   - Índices básicos si son necesarios
   - Evitar redundancia

Entrega solo el script SQL limpio y listo para ejecutar en Supabase.

**Respuesta:**

```sql
-- ============================================================
-- MentorLink – Database Schema
-- PostgreSQL / Supabase
-- ============================================================

-- Enable UUID generation
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- ============================================================
-- TABLE: perfiles
-- ============================================================
CREATE TABLE perfiles (
    id          UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    nombre      VARCHAR(150)  NOT NULL,
    bio         TEXT,
    rol         VARCHAR(20)   NOT NULL CHECK (rol IN ('mentor', 'estudiante')),
    created_at  TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

-- ============================================================
-- TABLE: sesiones
-- ============================================================
CREATE TABLE sesiones (
    id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    mentor_id       UUID        NOT NULL REFERENCES perfiles(id) ON DELETE CASCADE,
    estudiante_id   UUID        NOT NULL REFERENCES perfiles(id) ON DELETE CASCADE,
    fecha           TIMESTAMPTZ NOT NULL,
    estado          VARCHAR(20) NOT NULL DEFAULT 'pendiente'
                                CHECK (estado IN ('pendiente', 'activa', 'completada', 'cancelada')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT sesion_mentor_estudiante_diferentes
        CHECK (mentor_id <> estudiante_id)
);

-- ============================================================
-- TABLE: mensajes
-- ============================================================
CREATE TABLE mensajes (
    id            UUID    PRIMARY KEY DEFAULT gen_random_uuid(),
    sesion_id     UUID    NOT NULL REFERENCES sesiones(id) ON DELETE CASCADE,
    remitente_id  UUID    NOT NULL REFERENCES perfiles(id) ON DELETE CASCADE,
    contenido     TEXT    NOT NULL CHECK (char_length(contenido) > 0),
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- INDEXES
-- ============================================================

-- sesiones: búsquedas frecuentes por mentor o estudiante
CREATE INDEX idx_sesiones_mentor_id     ON sesiones(mentor_id);
CREATE INDEX idx_sesiones_estudiante_id ON sesiones(estudiante_id);
CREATE INDEX idx_sesiones_estado        ON sesiones(estado);

-- mensajes: recuperar mensajes de una sesión en orden cronológico
CREATE INDEX idx_mensajes_sesion_id    ON mensajes(sesion_id);
CREATE INDEX idx_mensajes_remitente_id ON mensajes(remitente_id);
CREATE INDEX idx_mensajes_created_at   ON mensajes(created_at);
```

---

**Decisiones de diseño clave:**

**UUIDs con `gen_random_uuid()`** — Supabase tiene `pgcrypto` disponible por defecto, por lo que no necesitas dependencias externas.

**Doble FK en `sesiones`** — Tanto `mentor_id` como `estudiante_id` apuntan a `perfiles`, lo que refleja que ambos roles viven en la misma tabla. La constraint `CHECK (mentor_id <> estudiante_id)` previene que un perfil se asigne como mentor y estudiante de sí mismo.

**`ON DELETE CASCADE`** — Si se elimina un perfil, sus sesiones se eliminan. Si se elimina una sesión, sus mensajes también. Esto mantiene integridad sin registros huérfanos.

**`TIMESTAMPTZ` en lugar de `TIMESTAMP`** — Almacena zona horaria, esencial para una plataforma con usuarios en múltiples regiones.

**Índices selectivos** — Se indexan las FKs (para JOINs rápidos), el campo `estado` (para filtros comunes como "sesiones activas") y `created_at` en mensajes (para ordenamiento cronológico en chats).
