# Guía del producto — EUIPO API Portal

Descripción funcional completa del portal y de los módulos custom, explicada de forma sencilla. Basada en el código de `eap/`, la documentación de `eap/docs/`, el contenido real del portal (`local-dev/fixtures/`) y un recorrido por el portal local con navegador (anónimo y como `developer`).

> Versión en inglés para la IA: `.ai/context/product-overview.md`. Detalle técnico de cada módulo: `.ai/skills/02`–`06`. Restricciones de plataforma y entrega: `.ai/skills/01-platform-and-delivery.md`.

---

## 1. Qué es el producto

Es el **escaparate y el autoservicio de las APIs de la EUIPO** (Oficina de Propiedad Intelectual de la Unión Europea). Está pensado para que empresas, despachos de abogados de marcas y desarrolladores conecten sus sistemas con la EUIPO para gestionar marcas y diseños: buscar una marca, presentar una solicitud, consultar el buzón de comunicaciones, etc.

**Aquí no se compra nada.** El acceso a las APIs es **gratuito**, tanto en Sandbox como en Producción (según la FAQ). El modelo es el siguiente:

- Las APIs se agrupan en **productos**.
- Cada producto tiene uno o más **planes**, que fijan los **límites de peticiones** (rate limits).
- El cliente **se suscribe** a un plan. La suscripción es un **permiso que EUIPO aprueba**, no una compra.

### Catálogo real (11 productos, cada uno con una sola API)

| Producto | Versión | Tags | Para qué sirve | Requiere aprobación |
|---|---|---|---|---|
| Trademark search | 1.1.0 | Trade mark | Buscar marcas en la base de datos de la EUIPO | Sí |
| Design search | 1.1.0 | Design | Buscar diseños | Sí |
| EUTM Filing | 1.1.0 | Application, Trade mark | **Presentar** solicitudes de marca de la UE | Sí |
| EUD Filing | 1.1.1 | Application, Design | **Presentar** solicitudes de diseño de la UE | Sí |
| Goods And Services | 1.2.0 | Classification, Trade mark | Taxonomía TMClass: buscar, validar y traducir productos y servicios | Sí |
| Product Indications | 1.0.0 | Classification, Design | Taxonomía DSClass: indicaciones de producto | Sí |
| Me | 1.2.0 | Trade mark, Design, Appeal… | Gestionar tu cartera de marcas y diseños | Sí |
| My EUIPO mailbox | 1.2.1 | … Communication | Buzón de comunicaciones del User Area | Sí |
| Document Repository | 1.0.0 | Attachment | Subir documentos temporales para adjuntarlos a los trámites | Sí |
| Persons | 1.0.0 | Applicant, Representative | Buscar solicitantes y representantes | Sí |
| Out of Commerce Works | 1.2.0 | Out of commerce works | Metadatos de obras fuera del circuito comercial | No |

Todos los productos tienen un único plan, **"Default Plan"**.

---

## 2. Quién hace qué

El portal es solo una de varias piezas:

```
                    ┌───────────────────────────┐
  Admin EUIPO ─────►│  IBM API Connect          │  (cerebro: define productos, planes,
                    │  (gestión)                │   límites, aprueba suscripciones,
                    └──────┬─────────────┬──────┘   emite client_id/secret)
          sincroniza el    │             │ configura
          catálogo         ▼             ▼
                 ┌──────────────┐   ┌─────────────────┐
  Desarrollador ►│ PORTAL       │   │ API Gateway     │◄── la app del cliente
  (navegador)    │ (Drupal/IBM) │   │ (portero)       │    llama a api.euipo.europa.eu
                 └──────┬───────┘   └────────┬────────┘
     eap_credentials_   │                    │ valida token
     sync registra la   ▼                    ▼
     app        ┌────────────────────────────────┐
                │ IdP de EUIPO (antes CAS, OIDC) │  (emite los tokens de acceso)
                └────────────────────────────────┘
```

| Pieza | Qué hace | ¿La controlamos? | En local la sustituye |
|---|---|---|---|
| **IBM API Connect** | Define productos, planes y límites; aprueba suscripciones; emite `client_id`/`client_secret`; sincroniza el catálogo con el portal | No | `mock-apic` + seed |
| **API Gateway** | Recibe cada llamada real a la API. Comprueba la suscripción, el token y el límite de peticiones (si se supera, devuelve **HTTP 429**) | No | No existe en local |
| **IdP de EUIPO** (antes CAS, ahora OIDC) | Autentica a las personas que entran al portal y emite los tokens OAuth2 que usan las apps | No | Keycloak |
| **Portal** (IBM API Developer Portal 10.0.8.12, Drupal 11) | Catálogo, documentación, alta de apps y peticiones de suscripción | Solo nuestros módulos y el tema | Se ejecuta el código real de IBM |

---

## 3. El recorrido del cliente, paso a paso

### Paso 0 — Tener una cuenta EUIPO y una organización

- La cuenta se crea en la web de la EUIPO, no en el portal; el autorregistro está desactivado.
- La persona tiene que pertenecer a una **organización consumidora** (su empresa) que existe en API Connect. Sin organización, al entrar ve "No permission".

Roles dentro de la organización:

| Rol | Puede |
|---|---|
| owner / administrator | Todo: organización, miembros, ajustes, apps, suscripciones |
| developer | Ver la organización y gestionar **apps y suscripciones** |
| viewer | Solo lectura (no puede crear apps) |

### Paso 1 — Explorar sin iniciar sesión

| Página | Ruta | Qué muestra |
|---|---|---|
| Portada | `/` → `/homepage` | Banner con 3 pasos: *Explore → Register, build & test → Apply* (bloques de `eap_apim_customs`) |
| API Products | `/product` | Catálogo con filtros por categoría, orden y paginación; tags en cada tarjeta |
| Ficha de producto | `/product/{alias}` | Descripción, sus APIs, descarga de las especificaciones y **planes**: candado = requiere aprobación, "2 rate limits" |
| Documentación de la API | `/product/{alias}/api/{api}` | Especificación OpenAPI renderizada con **Redoc** (`eap_redoc`): endpoints, parámetros, scopes y descarga |
| Getting started, Security, FAQ, Problems registry | páginas de contenido | Guías y catálogo de errores (los "problem types" de las APIs) |
| Status | `/status` | Monitor de PRTG en un iframe (`eap_api_status`) |
| Sandbox | enlace externo | Solo aparece en Producción y apunta al portal Sandbox |

**Límites de peticiones** en la ficha del producto, desplegando "2 rate limits":

| Límite | Valor en local | Tipo |
|---|---|---|
| default | 100 llamadas / segundo | soft |
| per-day | 10.000 llamadas / día | hard |

> Los valores locales son de ejemplo y los pone `seed.php`. Los reales están en API Connect. Si se supera un límite, el Gateway devuelve **429**.

### Paso 2 — Iniciar sesión

- Hay un solo botón, **"EUIPO Account"**: `eap_apim_customs` oculta las demás opciones.
- Recorrido: portal → API Connect (consumer-api) → IdP de EUIPO → vuelta al portal.
- En el primer inicio de sesión, API Connect informa de las organizaciones y roles del usuario, y el portal crea el usuario de Drupal.
- Tras entrar aparecen **Apps** en el menú y el nombre de la organización.
- Hay un límite de sesiones simultáneas por usuario (módulo `session_limit`).
- El administrador de Drupal (uid 1) entra por `/user/login?registry_url=/admin`.

### Paso 3 — Registrar una aplicación (`/application/new`)

| Campo | Notas |
|---|---|
| Title | Obligatorio |
| Description | Opcional |
| Application OAuth Redirect URL(s) | Necesarias para el flujo *authorization code*. El conjunto de URLs no puede ser **idéntico** al de otra app (validación de `eap_credentials_sync`) |

Qué ocurre al guardar:

1. API Connect genera el **API Key (`client_id`)** y el **API Secret (`client_secret`)**. El secret solo se muestra una vez; si se pierde, se puede **resetear**.
2. **`eap_credentials_sync`** registra esas mismas credenciales en el IdP de EUIPO. Sin este paso, la app tendría clave pero no podría conseguir tokens.

### Paso 4 — Suscribirse a un plan

1. En la ficha del producto, pulsar **Select** en el plan.
2. Elegir la aplicación (o crear una nueva).
3. Marcar la casilla **"acepto las condiciones de uso"** (añadida por `eap_apim_customs`). Sin marcarla, el botón no se activa.
4. Confirmar.

| Entorno | Qué pasa tras confirmar |
|---|---|
| Local (mock) | Suscripción activa al instante ("Subscription Complete") |
| Sandbox | Pendiente de aprobación (aproximadamente 1 día) |
| Producción | Pendiente de aprobación (hasta 1 semana). Además hay que enviar documentación a `docs.apiplatform@euipo.europa.eu`: DNI o pasaporte (y justificante de domicilio) si es persona física; DNI de quien actúa más un extracto del registro mercantil si es empresa |

Por norma, cada organización tiene **una sola suscripción por API**; las excepciones se estudian caso por caso.

### Paso 5 — Usar la API (fuera del portal)

1. **Pedir un token** al IdP con el `client_id` y el `client_secret`:

   | Flujo | Identifica | Cuándo se usa | Ejemplo |
   |---|---|---|---|
   | `client_credentials` | Solo la app | La API no necesita saber quién es el usuario final | Trademark search (scope `uid`) |
   | `authorization code` | App + usuario final (que da su consentimiento) | Actuar en nombre de alguien | EUTM Filing (scope `eutm-filing.application.write`) |

2. **Llamar a la API** en `https://api.euipo.europa.eu/{api}` con las cabeceras:
   - `X-IBM-Client-Id: <client_id>`
   - `Authorization: Bearer <access_token>`
3. **El Gateway comprueba** la suscripción, el token y el límite de peticiones.

| Dato | Valor |
|---|---|
| Duración del access token | 2 horas |
| Duración del refresh token | 4 horas |
| Si se supera el límite del plan | HTTP 429 |
| Token de Sandbox usado en Producción (o al revés) | HTTP 401: cada entorno tiene su propio IdP |
| Paginación | Empieza en la página 0 |

### Paso 6 — Sandbox antes que Producción

| | Sandbox | Producción |
|---|---|---|
| Datos | De prueba, realistas | Reales |
| APIs | Las mismas | Las mismas |
| IdP | Propio (`auth-sandbox.euipo.europa.eu`) | Propio |
| Requisito de acceso | Cuenta EUIPO + aprobación | Haber completado las pruebas en Sandbox + documentación + aprobación |
| Nombre del sitio | "Sandbox environment" | "API Portal" |
| Ajuste `is_sandbox` | Activado | Desactivado (se muestra el enlace "Sandbox" en el menú) |

Los **mismos paquetes** de módulos y tema se instalan en los dos portales; las diferencias se activan con ajustes.

---

## 4. Los módulos custom

> **Idea clave:** el portal es un producto de IBM que no podemos tocar. Nuestros módulos funcionan como **post-its pegados encima**: se enganchan a momentos concretos ("cuando se crea una app…", "cuando se pinta una página…") y añaden o cambian algo, sin modificar el código de IBM.

| Módulo | En una frase |
|---|---|
| `eap_credentials_sync` | Avisa al IdP cada vez que una app cambia, con registro y reintentos |
| `eap_apim_customs` | Pequeños retoques al portal de IBM, más un botón que reconstruye la portada y los bloques fijos |
| `eap_redoc` | Muestra la documentación de las APIs con Redoc y enlaza cada API con su guía |
| `eap_api_status` | Página `/status` que muestra el monitor de PRTG en un iframe |

### 4.1 `eap_credentials_sync` — el "cartero" de credenciales

**Problema:** API Connect da a la app un `client_id` y un `client_secret`, pero el IdP de EUIPO (que es quien emite los tokens) no se entera. Sin aviso, la app tiene credenciales que no sirven para conseguir un token.

**Solución:** cada vez que pasa algo con una app, el módulo apunta una "carta" y se la lleva al IdP.

#### Paso a paso

1. **El desarrollador hace algo con su app.** IBM lanza un "aviso" (hook) y el módulo lo escucha:

   | El usuario… | Hook de IBM | Operación | Datos que se envían |
   |---|---|---|---|
   | Crea una app | `apic_app_create` | `create` | nombre, redirect URLs, client_id, **secret** |
   | Edita la app | `apic_app_update` | `update` | nombre, redirect URLs, client_id (sin secret) |
   | Borra la app | `apic_app_pre_delete` | `delete` | client_id |
   | Resetea el secret | `apic_app_clientsecret_reset` | `reset` | nombre, redirect URLs, client_id, **secret nuevo** |

2. **Apunta la carta antes de enviarla.** `SyncEventRecorder` guarda un `sync_event` en la base de datos con estado **`pending`** y los datos. Si el envío falla, no se pierde nada.
3. **Intenta enviarla en ese mismo momento** (solo si "processing enabled" está activado):
   - `OAuthTokenProvider` pide un token al IdP (`client_credentials`).
   - `OidcHttpClient` llama al IdP:

     | Operación | Petición |
     |---|---|
     | create | `POST {endpoint}` |
     | update | `PUT {endpoint}/{clientId}` |
     | delete | `DELETE {endpoint}/{clientId}` |
     | reset | `DELETE` y después `POST` con el secret nuevo |

   - Si la respuesta es HTTP < 400, se marca **`success`**. Si no, **`failure`**, y se guarda el motivo (p. ej. `HTTP 409 on POST https://…: client already exists`).
4. **El cron reintenta** los eventos que no están en `success` mientras les queden intentos (5 por defecto).
5. **El cron limpia** los eventos `success` con más de 30 días.
6. **Revisión manual** en `/admin/reports/sync-events`: lista con filtros y, en el detalle de cada evento, el botón **Synchronize**, que lo reenvía aunque haya agotado los intentos.

**Extra:** al crear o editar una app se comprueba que el conjunto de redirect URLs no sea igual al de otra app, porque el IdP lo rechazaría. Si coincide, muestra "Invalid redirect URL."

```
Usuario crea app ─► IBM avisa ─► guardamos "pending" ─► enviamos al IdP
                                                         ├─ OK    → success
                                                         └─ error → failure ─► el cron reintenta
                                                                               └─ o un admin pulsa "Synchronize"
```

| Estado | Significado |
|---|---|
| `pending` | Registrado, todavía no enviado (o el procesamiento está desactivado) |
| `success` | El IdP lo aceptó |
| `failure` | Falló. El cron lo reintenta hasta `max_attempts`; después solo se reintenta con *Synchronize* |

**Ajustes** (`/admin/config/eap_credentials_sync/settings`): `applications_endpoint`, `auth_token_endpoint`, `auth_client_id`, `auth_client_secret`, `auth_scope`, `processing_enabled`, `cron_batch_size` (50), `max_attempts` (5), `success_retention_days` (30). Incluye los botones *Import Sync Events config* y *Delete all sync events*.

### 4.2 `eap_apim_customs` — la "caja de retoques"

Cambios pequeños que EUIPO necesita y que IBM no ofrece:

| # | Retoque | Cómo funciona |
|---|---|---|
| 1 | Casilla de condiciones al suscribirse | Añade un checkbox al asistente de suscripción; el botón no se activa hasta marcarlo. El enlace al PDF es un ajuste |
| 2 | Login simplificado | Si `all_register_options` está desactivado: texto "Sign in with your EUIPO account", oculta las demás formas de login y desactiva el autorregistro |
| 3 | Enlace "Sandbox" en el menú | Solo si el portal **no** es sandbox (es decir, en Producción) |
| 4 | Tags en las tarjetas de producto | Muestra los tags encima de la descripción |
| 5 | Índice en las guías | JS que genera el índice (TOC) en las páginas |
| 6 | Portada y bloques fijos | Botón **"Refresh static content"** |

**Qué hace "Refresh static content"** (`/admin/config/eap_apim_customs/refresh_site`):

1. Pone `/homepage` como portada. Es una página vacía; su contenido lo ponen los bloques.
2. Borra los bloques de branding y footer que Drupal copia del tema base.
3. Por cada bloque, **borra el existente y lo vuelve a crear** a partir de las plantillas Twig de `content/blocks/`:

   | Bloque | Región | Visible en |
   |---|---|---|
   | `eap_site_branding` | top_header | Todas las páginas |
   | `eap_main_banner` | content | Portada |
   | `eap_steps_banner` | content | Portada |
   | `eap_products_banner` | content | `/product` |
   | `eap_footer` | footer | Todas menos la documentación de las APIs |

> ⚠️ Si alguien edita esos bloques a mano en el entorno, el siguiente refresh **borra esos cambios**.

**Por qué:** los bloques viven en la base de datos y se pierden si se reinstala el tema. Con el botón, el diseño se reconstruye igual en cualquier entorno.

**Ajustes** (`/admin/config/eap_apim_customs/settings`):

| Ajuste | Efecto |
|---|---|
| `is_sandbox` | Modo Sandbox: sin enlace "Sandbox", banner de productos distinto |
| `all_register_options` | Mostrar todas las opciones de login y permitir el autorregistro (desactivado en Producción) |
| `data_protection_link` | Enlace "Data protection" del footer |
| `product_use_assessment_link` | Enlace "Conditions of use" (casilla de suscripción y footer) |

### 4.3 `eap_redoc` — el "traductor" de la documentación

**Problema:** el visor de APIs de IBM se consideraba poco claro. **Solución:** se reutiliza la página de IBM, pero el contenido lo pinta **Redoc**.

**A) Cuando alguien abre la página de una API:**
1. El módulo coge la especificación OpenAPI que IBM ya tiene guardada (campo `api_swagger`).
2. Se la pasa al navegador (`drupalSettings.eap_api`).
3. `eap_redoc.js` arranca Redoc dentro del contenedor de IBM (`.explorerWrapper`).
4. Si existe una guía que apunte a esa API, añade las pestañas **Reference / Guide**.

**B) Cuando alguien abre una guía enlazada a una API** (campos `apiref` y `prodref`):
1. Construye una cabecera con la ruta de navegación (Products › Producto › API), la versión, el estado y las pestañas Reference / Guide.
2. Si la guía está escrita en Markdown, le añade la clase `markdown-guide`, que es la que usa el índice de `eap_apim_customs`.

Redoc va **dentro del módulo** (`redoc.standalone.js`), así que no depende de ningún CDN.

### 4.4 `eap_api_status` — la "ventana" al monitor

1. Crea la página `/status` y el enlace "Status" en el menú principal.
2. Muestra el título "API Status", una leyenda de colores (Running / Warning / Error / Maintenance) y un **iframe** con el mapa público de PRTG.
3. La URL se configura en `/admin/config/eap_api_status/settings`.

El portal no comprueba nada por sí mismo; solo **muestra** lo que ya mide PRTG.

---

## 5. Qué pretende este proyecto (el repo)

No es el portal de producción. Es una **réplica local que se comporta como el portal real**, porque EUIPO no tiene acceso al servidor de IBM.

| Objetivo | Cómo |
|---|---|
| Desarrollar y probar las personalizaciones de EUIPO | 4 módulos en `eap/modules/` + el tema `eap_theme` en `eap/themes/`, montados en el sitio local |
| Simular todo lo externo | Keycloak = IdP de EUIPO · `mock-apic` = API Connect (incluido el registro de clientes) · seed = catálogo y contenido reales de `dev.euipo.europa.eu` |
| Entregar | `eap/scripts/build.sh` → subida por la interfaz de administración → portal de test (API Connect e IdP reales) → Sandbox → Producción |

En resumen, el portal de IBM aporta el catálogo, las apps y las suscripciones. Este proyecto le pone la marca EUIPO y añade lo que falta para su caso: que las credenciales lleguen al IdP de EUIPO, las condiciones de uso al suscribirse, la documentación con Redoc, la página de estado y el cambio entre Sandbox y Producción.

### Limitaciones del entorno local

| Limitación | Consecuencia |
|---|---|
| No hay API Gateway | El *Try it* llamaría a los servicios de producción; los límites de peticiones no se aplican |
| Suscripciones aprobadas al instante | No se puede probar el flujo de aprobación |
| Sin analytics ni facturación | Esos endpoints devuelven 404 o 501 |
| Sin webhooks | Los cambios del catálogo se aplican con el seed, no llegan en vivo |
| Los límites de los planes son de ejemplo | 100/s y 10.000/día los pone `seed.php` |

### Usuarios de prueba (contraseña = nombre de usuario)

| Usuario | Rol en la organización | Para probar |
|---|---|---|
| `admin` | — (uid 1, administrador de Drupal) | Administración y ajustes de los módulos |
| `developer` | owner | Flujo completo: apps, credenciales, suscripciones, organización |
| `orgadmin` | administrator | Administración de la organización |
| `orgdev` | developer | Crear apps y suscribirse |
| `orgviewer` | viewer | Solo lectura |
| `editor` | viewer + `content_author` | Editar contenido (páginas, guías) |
| `newdev` | ninguna | Usuario sin organización → "No permission" |
