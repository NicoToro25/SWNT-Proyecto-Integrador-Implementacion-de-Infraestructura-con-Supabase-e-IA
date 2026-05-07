# SWNT-Proyecto-Integrador-Implementación-de-Infraestructura-con-Supabase-e-IA

## Nicolás Toro Criollo

---

Scaffolding de aplicación de gestión de inventario sostenible utilizando React(vite) y Supabase. Se aplica el Desarrollo Basado en Especificaciones (BDD) para crear el scaffolding y garantizar que la implementación cumpla con los requisitos definidos en la fase de diseño. 

## Entregables

a. Captura de pantalla/Texto del Prompt Maestro

## Bitácora de prompts

### Fase 1 - Diseño del Esquema de Datos con IA

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

### Fase 1 - Iteración para mejorar respuesta

**Prompt:**

Revisa el anterior script SQL y mejóralo como un experto en PostgreSQL enfocado en producción en Supabase:

Quiero que:
- Optimices índices
- Verifiques integridad referencial
- Sugieras mejoras de rendimiento
- Asegures compatibilidad con Supabase Auth (auth.users)
- Mejores naming si es necesario
Devuelve el SQL final optimizado.

**Respuesta:**

```sql
-- ============================================================
-- MentorLink – Production Schema
-- PostgreSQL / Supabase  (rev 2)
-- ============================================================

-- ============================================================
-- TYPES
-- ============================================================
CREATE TYPE perfil_rol    AS ENUM ('mentor', 'estudiante');
CREATE TYPE sesion_estado AS ENUM ('pendiente', 'activa', 'completada', 'cancelada');

-- ============================================================
-- TABLE: perfiles
-- Linked 1-to-1 with Supabase Auth (auth.users)
-- ============================================================
CREATE TABLE public.perfiles (
    id          UUID          PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    nombre      VARCHAR(150)  NOT NULL,
    bio         TEXT,
    rol         perfil_rol    NOT NULL,
    created_at  TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

-- ============================================================
-- TABLE: sesiones
-- ============================================================
CREATE TABLE public.sesiones (
    id              UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    mentor_id       UUID          NOT NULL REFERENCES public.perfiles(id) ON DELETE RESTRICT,
    estudiante_id   UUID          NOT NULL REFERENCES public.perfiles(id) ON DELETE RESTRICT,
    fecha           TIMESTAMPTZ   NOT NULL,
    estado          sesion_estado NOT NULL DEFAULT 'pendiente',
    created_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW(),

    CONSTRAINT sesiones_roles_distintos
        CHECK (mentor_id <> estudiante_id)
);

-- ============================================================
-- TABLE: mensajes
-- ============================================================
CREATE TABLE public.mensajes (
    id            UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    sesion_id     UUID        NOT NULL REFERENCES public.sesiones(id) ON DELETE CASCADE,
    remitente_id  UUID        NOT NULL REFERENCES public.perfiles(id) ON DELETE RESTRICT,
    contenido     TEXT        NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT mensajes_contenido_no_vacio
        CHECK (char_length(TRIM(contenido)) > 0)
);

-- ============================================================
-- FUNCTION + TRIGGER: auto-update updated_at
-- ============================================================
CREATE OR REPLACE FUNCTION public.set_updated_at()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_perfiles_updated_at
    BEFORE UPDATE ON public.perfiles
    FOR EACH ROW EXECUTE FUNCTION public.set_updated_at();

CREATE TRIGGER trg_sesiones_updated_at
    BEFORE UPDATE ON public.sesiones
    FOR EACH ROW EXECUTE FUNCTION public.set_updated_at();

-- ============================================================
-- FUNCTION + TRIGGER: validate mentor/estudiante roles
-- ============================================================
CREATE OR REPLACE FUNCTION public.validar_roles_sesion()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    rol_mentor     perfil_rol;
    rol_estudiante perfil_rol;
BEGIN
    SELECT rol INTO rol_mentor     FROM public.perfiles WHERE id = NEW.mentor_id;
    SELECT rol INTO rol_estudiante FROM public.perfiles WHERE id = NEW.estudiante_id;

    IF rol_mentor IS DISTINCT FROM 'mentor' THEN
        RAISE EXCEPTION 'El perfil mentor_id (%) no tiene rol mentor.', NEW.mentor_id;
    END IF;

    IF rol_estudiante IS DISTINCT FROM 'estudiante' THEN
        RAISE EXCEPTION 'El perfil estudiante_id (%) no tiene rol estudiante.', NEW.estudiante_id;
    END IF;

    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_sesiones_validar_roles
    BEFORE INSERT OR UPDATE ON public.sesiones
    FOR EACH ROW EXECUTE FUNCTION public.validar_roles_sesion();

-- ============================================================
-- FUNCTION + TRIGGER: validate sender belongs to session
-- ============================================================
CREATE OR REPLACE FUNCTION public.validar_remitente_sesion()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    v_mentor_id     UUID;
    v_estudiante_id UUID;
BEGIN
    SELECT mentor_id, estudiante_id
      INTO v_mentor_id, v_estudiante_id
      FROM public.sesiones
     WHERE id = NEW.sesion_id;

    IF NEW.remitente_id NOT IN (v_mentor_id, v_estudiante_id) THEN
        RAISE EXCEPTION
            'El remitente (%) no pertenece a la sesión (%).',
            NEW.remitente_id, NEW.sesion_id;
    END IF;

    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_mensajes_validar_remitente
    BEFORE INSERT ON public.mensajes
    FOR EACH ROW EXECUTE FUNCTION public.validar_remitente_sesion();

-- ============================================================
-- INDEXES
-- ============================================================

-- perfiles
CREATE INDEX idx_perfiles_rol ON public.perfiles(rol);

-- sesiones
CREATE INDEX idx_sesiones_mentor_id      ON public.sesiones(mentor_id);
CREATE INDEX idx_sesiones_estudiante_id  ON public.sesiones(estudiante_id);
CREATE INDEX idx_sesiones_estado_fecha   ON public.sesiones(estado, fecha DESC);

-- Partial index: active sessions only (high-frequency query)
CREATE INDEX idx_sesiones_activas
    ON public.sesiones(mentor_id, estudiante_id)
    WHERE estado IN ('pendiente', 'activa');

-- mensajes
CREATE INDEX idx_mensajes_sesion_created
    ON public.mensajes(sesion_id, created_at ASC);

CREATE INDEX idx_mensajes_remitente_id
    ON public.mensajes(remitente_id);

-- ============================================================
-- ROW LEVEL SECURITY (RLS)
-- ============================================================
ALTER TABLE public.perfiles  ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.sesiones  ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.mensajes  ENABLE ROW LEVEL SECURITY;

-- perfiles: each user manages only their own profile
CREATE POLICY perfiles_select ON public.perfiles
    FOR SELECT USING (true);

CREATE POLICY perfiles_insert ON public.perfiles
    FOR INSERT WITH CHECK (id = auth.uid());

CREATE POLICY perfiles_update ON public.perfiles
    FOR UPDATE USING (id = auth.uid());

CREATE POLICY perfiles_delete ON public.perfiles
    FOR DELETE USING (id = auth.uid());

-- sesiones: visible only to participants
CREATE POLICY sesiones_select ON public.sesiones
    FOR SELECT USING (
        auth.uid() IN (mentor_id, estudiante_id)
    );

CREATE POLICY sesiones_insert ON public.sesiones
    FOR INSERT WITH CHECK (
        auth.uid() IN (mentor_id, estudiante_id)
    );

CREATE POLICY sesiones_update ON public.sesiones
    FOR UPDATE USING (
        auth.uid() IN (mentor_id, estudiante_id)
    );

-- mensajes: visible only to session participants
CREATE POLICY mensajes_select ON public.mensajes
    FOR SELECT USING (
        EXISTS (
            SELECT 1 FROM public.sesiones s
             WHERE s.id = sesion_id
               AND auth.uid() IN (s.mentor_id, s.estudiante_id)
        )
    );

CREATE POLICY mensajes_insert ON public.mensajes
    FOR INSERT WITH CHECK (
        remitente_id = auth.uid()
        AND EXISTS (
            SELECT 1 FROM public.sesiones s
             WHERE s.id = sesion_id
               AND auth.uid() IN (s.mentor_id, s.estudiante_id)
        )
    );

-- ============================================================
-- AUTO-CREATE perfil on new auth.users signup
-- ============================================================
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER
LANGUAGE plpgsql
SECURITY DEFINER SET search_path = public
AS $$
BEGIN
    INSERT INTO public.perfiles (id, nombre, rol)
    VALUES (
        NEW.id,
        COALESCE(NEW.raw_user_meta_data->>'nombre', NEW.email),
        (NEW.raw_user_meta_data->>'rol')::perfil_rol
    );
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_on_auth_user_created
    AFTER INSERT ON auth.users
    FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();
```

---

**Cambios respecto a la versión anterior:**

**ENUMs nativos** — `perfil_rol` y `sesion_estado` reemplazan los `VARCHAR + CHECK`. Son más eficientes en almacenamiento, auto-documentados y evitan valores inválidos a nivel de tipo, no solo de constraint.

**`perfiles.id` = `auth.users.id`** — Relación 1-a-1 directa con Supabase Auth. Elimina la necesidad de una columna `user_id` separada y garantiza que no existan perfiles sin usuario real.

**`ON DELETE RESTRICT` en sesiones** — En producción es más seguro que `CASCADE`: impide eliminar un perfil si tiene sesiones activas, forzando una decisión explícita en la lógica de negocio.

**Triggers de validación de roles** — Un `CHECK` no puede hacer JOINs entre tablas. El trigger `trg_sesiones_validar_roles` garantiza en base de datos que `mentor_id` siempre tenga rol `mentor` y `estudiante_id` siempre tenga rol `estudiante`, cerrando una brecha de integridad que el esquema anterior dejaba abierta.

**Trigger de remitente** — Previene que un usuario ajeno a la sesión inyecte mensajes, incluso si RLS fallara o se llamara directamente por SQL.

**Índice parcial de sesiones activas** — `WHERE estado IN ('pendiente', 'activa')` crea un índice pequeño y rápido para la consulta más frecuente del sistema, sin indexar sesiones archivadas.

**RLS completo** — Cada tabla tiene políticas por operación. Los mensajes validan participación en sesión tanto en `SELECT` como en `INSERT`, sin depender solo del `remitente_id`.

**`handle_new_user`** — Crea el perfil automáticamente al registrarse en Supabase Auth, leyendo `nombre` y `rol` desde `raw_user_meta_data` que el cliente envía en el signup.

### Fase 2 - Configuración de Seguridad (RLS)

**Prompt:**

Actúa como experto en seguridad en Supabase (PostgreSQL + Row Level Security).

Tengo estas tablas:

- perfiles
- sesiones
- mensajes

Necesito políticas RLS para que:

1. Un usuario solo pueda ver:
   - Sus propias sesiones (como mentor o estudiante)
   - Sus propios mensajes (como remitente)

2. Un usuario solo pueda insertar:
   - Mensajes donde sea el remitente
   - Sesiones donde participe

3. Consideraciones:
   - Usa auth.uid()
   - Activa RLS en todas las tablas
   - Define políticas claras y separadas por acción (SELECT, INSERT)

Devuelve:
- ALTER TABLE para activar RLS
- CREATE POLICY para cada caso

**Respuesta:**

```sql
-- ============================================================
-- MentorLink – Row Level Security (Production)
-- PostgreSQL / Supabase
-- ============================================================

-- ============================================================
-- 1. ACTIVAR RLS EN TODAS LAS TABLAS
-- ============================================================
ALTER TABLE public.perfiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.sesiones ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.mensajes ENABLE ROW LEVEL SECURITY;

-- Forzar RLS incluso para el propietario de la tabla
ALTER TABLE public.perfiles  FORCE ROW LEVEL SECURITY;
ALTER TABLE public.sesiones  FORCE ROW LEVEL SECURITY;
ALTER TABLE public.mensajes  FORCE ROW LEVEL SECURITY;

-- ============================================================
-- 2. HELPER FUNCTIONS (Security Definer)
-- Encapsulan lógica reutilizable y evitan repetir subqueries
-- en cada política → mejor rendimiento y mantenibilidad
-- ============================================================

-- ¿El usuario autenticado participa en esta sesión?
CREATE OR REPLACE FUNCTION public.usuario_en_sesion(p_sesion_id UUID)
RETURNS BOOLEAN
LANGUAGE sql
STABLE
SECURITY DEFINER
SET search_path = public
AS $$
    SELECT EXISTS (
        SELECT 1
          FROM public.sesiones
         WHERE id = p_sesion_id
           AND auth.uid() IN (mentor_id, estudiante_id)
    );
$$;

-- ¿El usuario autenticado tiene el rol indicado?
CREATE OR REPLACE FUNCTION public.usuario_tiene_rol(p_rol perfil_rol)
RETURNS BOOLEAN
LANGUAGE sql
STABLE
SECURITY DEFINER
SET search_path = public
AS $$
    SELECT EXISTS (
        SELECT 1
          FROM public.perfiles
         WHERE id   = auth.uid()
           AND rol  = p_rol
    );
$$;

-- ============================================================
-- 3. TABLA: perfiles
-- ============================================================

-- Cualquier usuario autenticado puede ver todos los perfiles
-- (necesario para buscar mentores / estudiantes disponibles)
CREATE POLICY "perfiles: lectura pública autenticada"
    ON public.perfiles
    FOR SELECT
    TO authenticated
    USING (true);

-- Solo puede crear su propio perfil (id debe coincidir con auth.uid)
CREATE POLICY "perfiles: insertar propio"
    ON public.perfiles
    FOR INSERT
    TO authenticated
    WITH CHECK (id = auth.uid());

-- Solo puede editar su propio perfil
CREATE POLICY "perfiles: actualizar propio"
    ON public.perfiles
    FOR UPDATE
    TO authenticated
    USING     (id = auth.uid())
    WITH CHECK (id = auth.uid());

-- Solo puede eliminar su propio perfil
CREATE POLICY "perfiles: eliminar propio"
    ON public.perfiles
    FOR DELETE
    TO authenticated
    USING (id = auth.uid());

-- ============================================================
-- 4. TABLA: sesiones
-- ============================================================

-- Solo ve sesiones donde participa como mentor o estudiante
CREATE POLICY "sesiones: ver propias"
    ON public.sesiones
    FOR SELECT
    TO authenticated
    USING (
        auth.uid() IN (mentor_id, estudiante_id)
    );

-- Solo puede crear sesiones donde él participa
-- Validación de roles delegada al trigger trg_sesiones_validar_roles
CREATE POLICY "sesiones: insertar como participante"
    ON public.sesiones
    FOR INSERT
    TO authenticated
    WITH CHECK (
        auth.uid() IN (mentor_id, estudiante_id)
    );

-- Solo los participantes pueden actualizar la sesión
-- (ej: cambiar estado a cancelada / completada)
CREATE POLICY "sesiones: actualizar como participante"
    ON public.sesiones
    FOR UPDATE
    TO authenticated
    USING     (auth.uid() IN (mentor_id, estudiante_id))
    WITH CHECK (auth.uid() IN (mentor_id, estudiante_id));

-- No se permite DELETE desde el cliente (solo via service_role)
-- Ausencia de política DELETE = denegación implícita ✓

-- ============================================================
-- 5. TABLA: mensajes
-- ============================================================

-- Solo ve mensajes de sesiones donde participa
CREATE POLICY "mensajes: ver de mis sesiones"
    ON public.mensajes
    FOR SELECT
    TO authenticated
    USING (
        public.usuario_en_sesion(sesion_id)
    );

-- Solo puede insertar mensajes:
--   a) donde él es el remitente
--   b) en sesiones donde participa
--   c) y la sesión esté activa (no completada ni cancelada)
CREATE POLICY "mensajes: insertar como remitente activo"
    ON public.mensajes
    FOR INSERT
    TO authenticated
    WITH CHECK (
        remitente_id = auth.uid()
        AND public.usuario_en_sesion(sesion_id)
        AND EXISTS (
            SELECT 1
              FROM public.sesiones
             WHERE id     = sesion_id
               AND estado = 'activa'
        )
    );

-- No se permite UPDATE ni DELETE de mensajes desde el cliente
-- Los mensajes son inmutables una vez enviados ✓

-- ============================================================
-- 6. SERVICE ROLE BYPASS (documentado, no requiere SQL)
-- ============================================================
-- El rol `service_role` de Supabase bypasea RLS por defecto.
-- Úsalo SOLO en:
--   - Edge Functions administrativas
--   - Migraciones de datos
--   - Tareas de mantenimiento server-side
--
-- NUNCA expongas la service_role key en el cliente.
-- ============================================================

-- ============================================================
-- 7. VERIFICACIÓN (ejecutar en Supabase SQL Editor)
-- ============================================================

-- Ver todas las políticas activas:
-- SELECT schemaname, tablename, policyname, cmd, qual, with_check
--   FROM pg_policies
--  WHERE schemaname = 'public'
--  ORDER BY tablename, cmd;

-- Simular sesión de un usuario específico:
-- SET LOCAL role = authenticated;
-- SET LOCAL request.jwt.claims = '{"sub": "UUID-DEL-USUARIO"}';
-- SELECT * FROM public.mensajes; -- debe retornar solo sus mensajes
```

---

**Arquitectura de seguridad por capa:**

```
Cliente (anon / authenticated)
        │
        ▼
┌───────────────────┐
│   Supabase Auth   │  auth.uid() identifica al usuario
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│   RLS Policies    │  Primera barrera: filtra filas por identidad
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│    Triggers       │  Segunda barrera: valida roles y participación
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  Base de datos    │  Solo llegan operaciones válidas y autorizadas
└───────────────────┘
```

**Decisiones clave de esta implementación:**

**`FORCE ROW LEVEL SECURITY`** — Sin esto, el propietario de la tabla (rol `postgres`) saltaría las políticas. En Supabase esto afecta las Edge Functions que usan el cliente estándar.

**Helper functions con `SECURITY DEFINER`** — Ejecutan con privilegios del creador, no del invocador. Esto permite hacer lookups internos sin exponer tablas intermedias y centraliza lógica que de otro modo se repetiría en cada política, reduciendo el riesgo de inconsistencias.

**Inmutabilidad de mensajes** — La ausencia de políticas `UPDATE` y `DELETE` en `mensajes` es intencional. En una plataforma de mentoría, el historial debe ser auditable. Si se necesita "borrar", se recomienda una columna `deleted_at` con soft delete.

**No hay `DELETE` en sesiones desde el cliente** — Las sesiones solo cambian de estado (`cancelada`, `completada`). El borrado físico, si aplica, debe hacerse desde `service_role` con lógica de negocio explícita.

**Política de inserción de mensajes valida `estado = 'activa'`** — Previene que se envíen mensajes a sesiones cerradas o aún no confirmadas, directamente en la base de datos sin depender del frontend.

### Fase 3 - Implementación con Supabase CLI

* Principalmente se instala Supabase CLI

   ```
   npm install -g supabase
   ```
* Se inicializa el proyecto Supabase local

   ```
   supabase init
   ```

   Este comando crea las carpetas supabase/migrations

* Se crea la migración

   ```
   supabase migration new inicializar_esquema
   ```

   Esto crea un archivo .sql dentro de supabase/migrations

* Dentro del archivo SQL generado, se pega el documento generado por la IA previamente.

* Se debe enlazar el proyecto con Supabase, para eso se necesita autenticarse y el siguiente comando.

   ```
   supabase link --project-ref TU_PROYECT_REF
   ```

* Finalmente, se aplica la migración

   ```
   supabase db push
   ```

### Fase 4 - Configuración de Autenticación

# Google OAuth en Supabase — Guía de Configuración

---

## Paso 1 — Crear credenciales en Google Cloud Console

Se sigue esta ruta en www.console.cloud.google.com:

```
APIs & Services → Credentials → Create Credentials → OAuth Client ID
```

Configura la pantalla de consentimiento primero si no lo has hecho:

```
OAuth consent screen
├── User Type       → External
├── App name        → MentorLink
├── Support email   → tu@email.com
└── Scopes          → email, profile, openid  ← mínimo necesario
```

Luego crea el Client ID:

```
Application type    → Web application
Name                → MentorLink Web Client

Authorized JavaScript origins
└── https://<tu-proyecto>.supabase.co

Authorized redirect URIs
└── https://<tu-proyecto>.supabase.co/auth/v1/callback
```

> La Redirect URI debe ser **exacta**. Un carácter de diferencia rompe el flujo OAuth.

Al guardar, Google entrega:

```
CLIENT_ID      →  xxxxxx.apps.googleusercontent.com
CLIENT_SECRET  →  GOCSPX-xxxxxxxxxxxxxxxxxxxxxx
```

Cópialos — los usarás en el Paso 2.

---

## Paso 2 — Habilitar Google Auth en Supabase Dashboard

```
supabase.com/dashboard
└── Tu proyecto
    └── Authentication
        └── Providers
            └── Google  →  Enable: ON
```

Pega las credenciales obtenidas en el Paso 1:

| Campo | Valor |
|---|---|
| Client ID | `xxxxxx.apps.googleusercontent.com` |
| Client Secret | `GOCSPX-xxxxxxxxxxxxxxxxxxxxxx` |
| Redirect URL | Se genera automáticamente ✓ |

La **Redirect URL** que muestra Supabase es la que debes usar en Google Cloud Console:

```
https://<tu-proyecto>.supabase.co/auth/v1/callback
```

Guarda los cambios. ✓

---

## Paso 3 — Implementar el flujo en el cliente

**Instalar el SDK:**
```bash
npm install @supabase/supabase-js
```

**Inicializar Supabase:**
```javascript
// lib/supabase.js
import { createClient } from '@supabase/supabase-js'

export const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY
)
```

**Login con Google:**
```javascript
// Redirige al usuario a Google → regresa autenticado
const { error } = await supabase.auth.signInWithOAuth({
  provider: 'google',
  options: {
    redirectTo: 'https://tu-app.com/auth/callback',  // tu app, no Supabase
    scopes: 'email profile'
  }
})
```

**Capturar la sesión al volver:**
```javascript
// app/auth/callback/route.js  (Next.js App Router)
import { supabase } from '@/lib/supabase'

export async function GET(request) {
  const { searchParams } = new URL(request.url)
  const code = searchParams.get('code')

  if (code) {
    await supabase.auth.exchangeCodeForSession(code)
  }

  return NextResponse.redirect('/dashboard')
}
```

**Leer el usuario autenticado:**
```javascript
const { data: { user } } = await supabase.auth.getUser()

console.log(user.id)        // UUID → coincide con perfiles.id
console.log(user.email)     // email de Google
console.log(user.user_metadata.full_name)   // nombre de Google
```

---

## Flujo completo

```
Usuario click "Login con Google"
        │
        ▼
supabase.auth.signInWithOAuth()
        │
        ▼
Google OAuth Consent Screen
        │
        ▼
Google → redirect a Supabase /auth/v1/callback
        │
        ▼
Supabase valida → crea sesión + JWT
        │
        ▼
Supabase → redirect a tu app /auth/callback
        │
        ▼
exchangeCodeForSession() → sesión activa
        │
        ▼
auth.uid() disponible en RLS ✓
```

---

## Variables de entorno requeridas

```bash
# .env.local
NEXT_PUBLIC_SUPABASE_URL=https://<tu-proyecto>.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI...

# Nunca en el cliente
SUPABASE_SERVICE_ROLE_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI...
```

> `SERVICE_ROLE_KEY` bypasea RLS. Solo en servidor, nunca expuesta al cliente.
