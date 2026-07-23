# Documentación — Recolección de firmas para clases virtuales

Sistema web para recolectar firmas digitales de estudiantes que apoyan una carta
dirigida al Decano de la **Universidad Privada Domingo Savio**, solicitando clases
virtuales por motivos climatológicos. Todas las firmas se suman a **un único
documento** que luego se exporta como PDF para presentarlo al Decanato.

No hay servidor propio: todo corre en el navegador y usa **Supabase** como backend
(base de datos + autenticación).

---

## 1. Arquitectura general

```
Estudiantes (index.html) ───POST firma────►  ┌──────────────────────┐
                                             │       Supabase       │
Organizador (panel_firmas.html) ◄──GET────── │  · tabla "firmas"    │
                            └───DELETE──────► │  · Auth (login)      │
                                             │  · RPC contar_firmas │
                     PDF final ◄─────────────┴──────────────────────┘
```

- **Frontend:** archivos HTML estáticos, **publicados en GitHub Pages**.
- **Backend:** Supabase (`https://ithfislojjtnbasvfrcy.supabase.co`).
- **Generación de PDF:** en el navegador, con `html2canvas` + `jsPDF` (cargados por CDN).

### URLs públicas (GitHub Pages)

El sitio se publica desde la rama `main` (carpeta raíz) del repositorio
`douglasDFH/clasesVirtuales`:

| Página | URL |
|--------|-----|
| Firmas (público) | `https://douglasdfh.github.io/clasesVirtuales/` |
| Panel del organizador | `https://douglasdfh.github.io/clasesVirtuales/panel_firmas.html` |
| Recuperar contraseña | `https://douglasdfh.github.io/clasesVirtuales/recuperar.html` |

Cada `git push` a `main` vuelve a desplegar el sitio automáticamente.

---

## 2. `index.html` — Página pública (estudiantes)

Es el enlace que se comparte con todos. Flujo:

1. El estudiante **lee la carta** de solicitud.
2. Escribe su **nombre completo** y **carnet de identidad (CI)**.
3. **Firma con el dedo o el mouse** sobre un `<canvas>`.
   - `sizeCanvas()` ajusta la resolución al `devicePixelRatio` para que la firma se vea nítida.
   - `firmaRecortada()` detecta el área realmente dibujada y la recorta (con un pequeño margen), para que la firma no salga diminuta dentro del recuadro en el PDF. Devuelve un PNG en base64 (`data:image/png`).
4. Al pulsar **Enviar**, valida (nombre ≥ 3 caracteres, CI ≥ 4, firma presente) y hace `POST /rest/v1/firmas` con `{ nombre, ci, firma }`.
5. `actualizarContador()` llama a la función `contar_firmas` y muestra "X personas ya firmaron".
6. Usa `localStorage['firma_enviada']` para recordar si ese dispositivo ya firmó y mostrar la pantalla de agradecimiento. El botón "Registrar otra firma" limpia ese estado para firmar con otros datos.

Clave usada: la **clave pública `anon`** (embebida en el HTML). Sirve solo para **insertar** firmas.

---

## 3. `panel_firmas.html` — Panel privado (organizador)

Panel administrativo protegido con **login de Supabase Auth** (correo + contraseña). Flujo:

1. **Login** (`POST /auth/v1/token?grant_type=password`). La sesión (`access_token` + `refresh_token`) se guarda en `localStorage['panel_sesion']` y se restaura al volver a abrir.
2. La función `api()` añade el token del organizador a cada llamada; si expira (HTTP 401) lo renueva con `refrescar()` y reintenta una vez.
3. **Cargar firmas:** `GET /rest/v1/firmas?select=*&order=creado_en.asc`.
4. `marcarDuplicados()` normaliza el CI (minúsculas, solo letras/números) y marca en amarillo los **carnets repetidos**, desmarcándolos para no incluirlos dos veces.
5. El organizador puede **marcar/desmarcar** cuáles incluir y **eliminar** firmas (`DELETE /rest/v1/firmas?id=eq.<id>`).
6. **Generar PDF** (`armarDocumento`):
   - Página 1: la carta + hasta **2 firmas** (`SLOTS_PAGE_1`).
   - Páginas siguientes: grilla de 2×5 = **10 firmas por hoja** (`SLOTS_PER_EXTRA_PAGE`).
   - Cada hoja se rasteriza con `html2canvas` (scale 2) y se agrega al PDF tamaño **carta** con `jsPDF`. Se descarga como `Solicitud_Clases_Virtuales_Firmada.pdf`.

---

## 3b. `recuperar.html` — Cambiar / recuperar la contraseña del organizador

La contraseña del organizador es la de su cuenta en **Supabase Auth** (no está en el
código). Para cambiarla cuando **se olvida** (no puedes entrar al panel):

1. En el panel de Supabase → **Authentication → Users**, se abre el usuario y se pulsa
   **"Send password recovery"** (envía un correo).
2. El correo trae un enlace que Supabase redirige a la **Site URL** configurada, con un
   token de recuperación en el fragmento (`#access_token=...&type=recovery`).
3. Ese enlace abre **`recuperar.html`**, que lee el token y muestra un formulario para
   escribir la contraseña nueva. Al guardar, hace `PUT /auth/v1/user` con el token.

### Configuración necesaria en Supabase (Authentication → URL Configuration)

Para que el correo de recuperación funcione (antes redirigía a `http://localhost:3000`
y daba **404**), se dejó configurado así:

| Campo | Valor |
|-------|-------|
| **Site URL** | `https://douglasdfh.github.io/clasesVirtuales/recuperar.html` |
| **Redirect URLs** | `https://douglasdfh.github.io/clasesVirtuales/**` |

> El token de recuperación dura **1 hora**. Si expira, se solicita un correo nuevo.

---

## 4. Modelo de datos (tabla `firmas`)

Campos deducidos del código:

| Columna     | Tipo      | Notas                                              |
|-------------|-----------|----------------------------------------------------|
| `id`        | `bigint`  | Identificador autoincremental (verificado).        |
| `nombre`    | texto     | Nombre completo del estudiante.                    |
| `ci`        | texto     | Carnet de identidad.                               |
| `firma`     | texto     | Imagen PNG de la firma en base64 (`data:image/png`).|
| `creado_en` | timestamp | Fecha de registro (usada para ordenar).            |

Además existe la función RPC **`contar_firmas`** (devuelve el total de firmas sin exponer las filas).

---

## 5. Verificación de seguridad (RLS)

> Pruebas realizadas el **2026-07-22** contra la API pública de Supabase usando la
> clave `anon` que está embebida en el HTML (es decir, simulando a un atacante
> anónimo). No se insertó ni se borró ningún dato real.

El diseño depende de que las políticas **Row Level Security (RLS)** de Supabase
permitan que el público solo **inserte** firmas, pero **no pueda leer ni borrar**
los datos de los demás (nombre, CI y firma son datos personales sensibles).

### Resultados

**A) Pruebas de caja negra contra la API (como anónimo, sin login):**

| # | Prueba (como anónimo, sin login)                          | Resultado                     | Estado |
|---|-----------------------------------------------------------|-------------------------------|--------|
| 1 | `SELECT *` sobre `firmas`                                  | HTTP 200 pero `[]` (vacío)    | ✅ Protegido |
| 2 | `contar_firmas` (RPC)                                      | HTTP 200 → **21**             | ✅ Esperado (público a propósito) |
| 3 | `SELECT nombre,ci`                                         | HTTP 200 pero `[]` (vacío)    | ✅ Protegido |
| 4 | `DELETE` con id inexistente                                | HTTP 200 `[]` (0 filas)       | ⚠️ No concluyente por API |

**B) Verificación directa de las políticas RLS en el panel de Supabase** (Database → Policies).
RLS está **activado** en la tabla `firmas` y existen exactamente estas 3 políticas:

| Política                | Comando  | Aplica al rol           | Significado                                  |
|-------------------------|----------|-------------------------|----------------------------------------------|
| `cualquiera puede firmar` | `INSERT` | `anon`                  | El público puede **insertar** firmas.        |
| `organizador_lee`         | `SELECT` | `authenticated`         | Solo el organizador (con login) puede **leer**. |
| `organizador_borra`       | `DELETE` | `authenticated`         | Solo el organizador (con login) puede **borrar**. |

No existe ninguna política de `DELETE`, `SELECT` ni `UPDATE` para el rol `anon`.

### Interpretación

- **La lectura anónima está correctamente bloqueada.** Esto es lo más importante:
  la RPC confirma que hay **21 firmas** en la tabla, pero un `SELECT` anónimo
  devuelve una lista **vacía**. Eso demuestra que existe una política RLS de
  `SELECT` que impide a usuarios no autenticados leer las firmas, los carnets y
  los nombres. **Los datos personales están protegidos.** ✅
- **La clave `anon` en el HTML no es un problema de seguridad.** Es pública por
  diseño; solo habilita el `INSERT` de firmas. Leer y borrar requiere iniciar
  sesión como organizador.
- **El contador es público a propósito** (solo revela un número, no datos).
- **DELETE anónimo → BLOQUEADO (confirmado en el panel).** La duda que había
  quedado con la prueba de API se cerró revisando las políticas RLS directamente:
  la única política de borrado es `organizador_borra`, que aplica solo al rol
  `authenticated`. **No existe** política de `DELETE` para `anon`, así que un
  anónimo con la clave pública **no puede borrar firmas**. ✅
- **No hay política de `UPDATE`**, por lo que nadie (ni anon ni el organizador)
  puede modificar una firma ya registrada — correcto para este caso.

**Conclusión general: el diseño de seguridad está completo y bien configurado.**
El público solo puede insertar firmas; leer y borrar requieren iniciar sesión como
organizador. Los datos personales (nombre, CI, firma) están protegidos.

### Nota

- **INSERT anónimo:** no se probó por API para no ensuciar la tabla real, pero la
  política `cualquiera puede firmar` (rol `anon`) confirma que está habilitado a
  propósito — es el funcionamiento normal de la página pública.

### Endurecimiento aplicado (2026-07-22)

Se ejecutó SQL en Supabase (editor SQL, rol `postgres`) para reforzar la tabla
`firmas`. Todo se verificó después de aplicarlo:

**1) Restricciones de tamaño y formato (CHECK constraints)** — confirmadas con
`pg_constraint`:

| Restricción     | Regla aplicada a cada firma nueva                     |
|-----------------|-------------------------------------------------------|
| `firma_formato` | La firma debe empezar con `data:image/png;base64,`.   |
| `firma_tam_max` | La firma no puede pasar de **400.000 caracteres**.    |
| `nombre_tam`    | Nombre entre **3 y 80** caracteres.                   |
| `ci_tam`        | Carnet entre **4 y 30** caracteres.                   |

Se agregaron con `NOT VALID` para que apliquen solo a firmas **nuevas** sin
revalidar las 21 existentes.

**2) Rate limiting (trigger `trg_limite_firmas`)** — confirmado con `pg_trigger`
(existe y está habilitado). Antes de cada inserción cuenta las firmas de los
últimos **10 segundos**; si ya hay **8 o más**, rechaza la inserción con un mensaje
de error. Frena inserciones masivas automáticas.

> **Alcance del rate limiting:** es un límite *global* (no por IP, porque Supabase
> free no expone la IP a nivel de tabla). Frena spam automático masivo, pero un
> atacante paciente que espacie las inserciones podría evadirlo. Para bloqueo por
> IP haría falta una Edge Function. Para una campaña de firmas puntual, el trigger
> global es suficiente. El umbral (8 cada 10 s) se puede ajustar.

**Hallazgo adicional:** la política de INSERT `cualquiera puede firmar` (rol `anon`)
**ya tenía** una condición `WITH CHECK` propia que valida el largo del nombre (y
más campos). Por eso, al probar inserciones inválidas con la clave `anon`, fueron
rechazadas (error `42501` de RLS) **antes** de llegar a las restricciones CHECK.
Las nuevas restricciones y el trigger funcionan como **segunda capa de defensa**;
el rate limiting es protección genuinamente nueva. Las inserciones legítimas de la
página siguen funcionando, porque una firma real cumple tanto la condición RLS como
las restricciones CHECK.

**Cómo revertirlo** (si alguna vez hiciera falta):

```sql
drop trigger if exists trg_limite_firmas on public.firmas;
drop function if exists public.limite_firmas();
alter table public.firmas
  drop constraint if exists firma_formato,
  drop constraint if exists firma_tam_max,
  drop constraint if exists nombre_tam,
  drop constraint if exists ci_tam;
```

### Recomendaciones pendientes (opcionales)

1. Si se quiere un rate limiting **por IP** real, migrar la inserción a una Supabase
   Edge Function que valide la IP de origen.
2. Añadir el mismo límite de tamaño del lado del cliente (`index.html`) para dar un
   mensaje más claro al usuario antes de enviar (mejora de experiencia, no de
   seguridad).

---

## 6. Cómo usarlo

1. **Compartir** el enlace de `index.html` con los estudiantes para que firmen.
2. El **organizador** entra a `panel_firmas.html`, inicia sesión, pulsa
   **Actualizar firmas**, revisa/limpia duplicados y pulsa **Generar PDF**.
3. Presentar el PDF descargado al Decanato.
