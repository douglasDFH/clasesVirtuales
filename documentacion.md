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

- **Frontend:** 2 archivos HTML estáticos (se pueden alojar en cualquier hosting o abrir localmente).
- **Backend:** Supabase (`https://ithfislojjtnbasvfrcy.supabase.co`).
- **Generación de PDF:** en el navegador, con `html2canvas` + `jsPDF` (cargados por CDN).

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

### Recomendaciones adicionales

1. Verificar en Supabase que RLS esté **activado** en la tabla `firmas` y que las
   políticas sean: `INSERT` para `anon`; `SELECT`/`UPDATE`/`DELETE` solo para el
   rol autenticado (`authenticated`).
2. Considerar un límite de tamaño para el campo `firma` (base64) y validación
   server-side, para evitar que alguien envíe cargas enormes con la clave `anon`.
3. Opcional: limitar la frecuencia de inserciones (rate limiting) para evitar spam
   de firmas falsas, ya que el `INSERT` es público.

---

## 6. Cómo usarlo

1. **Compartir** el enlace de `index.html` con los estudiantes para que firmen.
2. El **organizador** entra a `panel_firmas.html`, inicia sesión, pulsa
   **Actualizar firmas**, revisa/limpia duplicados y pulsa **Generar PDF**.
3. Presentar el PDF descargado al Decanato.
