# Inspira – Crear Usuario con Área

**Versión:** 1.0 · **Autor:** Cristian · **Archivo único:** `inspira-create-user-area.php`

> API REST con token Bearer que crea un usuario de WordPress y lo inscribe en un área de Inspira, heredando la fecha de expiración del área.

## ¿Qué es?

Es un plugin pequeño (un solo archivo) que añade un **endpoint REST** al sitio de Inspira. Sirve para que un sistema externo, como WMS (wmslatam.com) o un flujo de n8n, haga dos cosas con una sola llamada:

1. Crear un **usuario de WordPress**.
2. Inscribirlo en un **área** de una empresa, creando su membresía en la tabla `{prefix}user_members`.

Es el endpoint al que se envían los usuarios dados de alta en otros sitios del ecosistema WMS para que también existan en Inspira con su área asignada.

---

## Endpoint

```
POST /wp-json/inspira/v1/create-user-area
Authorization: Bearer <INSPIRA_API_TOKEN>
Content-Type: application/json
```

### Body

```json
{
  "email": "correo@dominio.com",
  "password": "unaClave123",
  "area_id": 5
}
```

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `email` | string | Sí | Correo del nuevo usuario. También se usa como `user_login`. |
| `password` | string | Sí | Contraseña en texto plano. WordPress la guarda cifrada. |
| `area_id` | int | Sí | ID del área en `{prefix}areas`. |

---

## Flujo paso a paso

1. **Autenticación.** El header `Authorization` debe empezar por `Bearer `. El token se compara con la constante `INSPIRA_API_TOKEN` usando `hash_equals()`, para que la comparación no revele información por el tiempo que tarda.
2. **Validación del body.** `email`, `password` y `area_id` son obligatorios, y el correo debe tener un formato válido.
3. **Búsqueda del área.** Se busca el `area_id` en `{prefix}areas`. Si no existe, el proceso se detiene.
4. **Fecha de expiración del área.**
   - Si está vacía o es `0000-00-00`, o no se puede interpretar como fecha, el proceso se detiene **sin crear el usuario**.
   - Si es válida, se convierte al formato **`dd-mm-aaaa`**.
5. **Correo duplicado.** Si ya existe un usuario con ese correo, el proceso se detiene sin crear nada.
6. **Creación del usuario** con `wp_insert_user()`:
   - `user_login` y `user_email`: el correo.
   - `user_pass`: la contraseña recibida.
   - `role`: `subscriber`.
   - `user_nicename`: la parte del correo antes de la @ más 5 letras aleatorias (por ejemplo, `juan-xkqrt`).
7. **Inscripción en el área.** Se inserta un registro en `{prefix}user_members`:

   | Columna | Valor |
   |---|---|
   | `area_id` | el recibido |
   | `user_id` | el del usuario recién creado |
   | `status` | `ACTIVO` |
   | `days` | `0` |
   | `fecha_final_membresia` | la fecha de expiración del área (`dd-mm-aaaa`) |

8. **Respuesta** con el resultado de la operación.

---

## Respuestas

### Éxito: `201 Created`
```json
{
  "success": true,
  "message": "Usuario y membresía creados correctamente",
  "user_id": 123,
  "user_nicename": "juan-xkqrt",
  "area_id": 5,
  "status": "ACTIVO",
  "fecha_final_membresia": "31-12-2026"
}
```

### Errores

| HTTP | Motivo |
|---|---|
| `401` | Falta el header `Authorization: Bearer …` |
| `403` | El token no es válido |
| `400` | Faltan `email`, `password` o `area_id`, o el correo no es válido |
| `404` | El área no existe |
| `409` | Ya existe un usuario con ese correo |
| `422` | El área no tiene fecha de expiración o la que tiene no es válida |
| `500` | `wp_insert_user()` falló, o el usuario se creó pero falló la membresía (en ese caso la respuesta incluye `user_id`) |

Todas las respuestas de error tienen la forma `{ "success": false, "message": "…" }`.

---

## Dependencias
- Tablas `{prefix}areas` (`id` y `fecha_expiracion`) y `{prefix}user_members`, del ecosistema de áreas de Inspira.
- WordPress con la API REST activa.

## Observaciones técnicas
- **Token escrito en el código:** `INSPIRA_API_TOKEN` está como constante en el archivo del plugin. Es mejor moverla a `wp-config.php` o a una opción, y no subir el archivo a repositorios públicos.
- **Requiere PHP 8.0 o superior:** usa `str_starts_with()`, que en PHP 7.x produce un error fatal.
- **No es atómico:** si falla la inserción de la membresía, el usuario queda creado sin área. No se hace rollback, aunque la respuesta incluye el `user_id` para poder corregirlo.
- **No comprueba el cupo ni el estado del área:** crea la membresía aunque el área esté llena o inactiva.
- No envía correos ni dispara webhooks. El sistema que hace la llamada se encarga de avisar al usuario.
