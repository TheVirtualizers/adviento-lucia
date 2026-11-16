# Calendario de Lucía: código listo para GitHub Pages

Incluye dos vistas: `index.html` para Lucía y `admin.html` para Manuel. El panel permite abrir los 24 días en vista previa, cambiar textos y fotos y elegir efectos. Cada guardado crea un commit en GitHub.

## Dónde va cada cosa

- **Repositorio de la web** (`adviento-lucia`): sube el contenido de `web-github-pages.zip`. Puede ser público. GitHub Pages publica únicamente `/docs`.
- **Repositorio de datos** (`adviento-lucia-datos`): crea este repositorio como **Private** y sube el contenido de `datos-privados.zip`. Contiene `calendar.json` y después las fotos dentro de `photos/`. No actives Pages aquí.
- **Servicio de acceso**: el archivo `backend/worker.mjs` se ejecuta en Cloudflare Workers. Comprueba la contraseña y la fecha y lee/escribe en el repositorio privado. No guarda las fotos ni los vales en una base de datos externa.

GitHub Pages solo sirve archivos; no puede ocultar claves ni verificar una contraseña de forma segura. Por eso hay un pequeño servicio aparte. El código no contiene tu contraseña ni un token de GitHub.

La separación en dos repositorios permite usar una web pública con datos privados. El servicio rechaza un repositorio de datos público: los días futuros no deben poder consultarse mirando sus archivos. Si alguien tiene acceso directo al repositorio privado, podrá leerlos; solo tú debes tener acceso a ese repositorio.

## 1. Sube los dos repositorios

1. En GitHub crea `adviento-lucia` y sube **el contenido** de `web-github-pages.zip`, no el ZIP. `docs`, `backend`, `src` y este README deben quedar en la raíz.
2. Crea otro repositorio, `adviento-lucia-datos`, con visibilidad **Private**.
3. Sube allí el contenido de `datos-privados.zip`. `calendar.json` debe quedar en la raíz, no dentro de otra carpeta. Su rama debe ser `main`.
4. En el repositorio de la web entra en **Settings → Pages → Build and deployment → Deploy from a branch**. Elige **main** y **/docs** y guarda.
5. Espera a que GitHub muestre la dirección de la web. Será similar a `https://TU_USUARIO.github.io/adviento-lucia/`.

No necesitas compilar nada para este primer paso: `/docs` ya incluye la web preparada. Aunque la página visual aparezca, aún falta conectar el servicio para abrir el calendario y entrar al panel.

## 2. Prepara el permiso de GitHub

En tu cuenta: **Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token**.

- Elige solo el repositorio privado `adviento-lucia-datos`.
- Dale permiso de repositorio **Contents: Read and write**. Metadata de lectura se incluye automáticamente.
- No necesita permisos sobre la web ni sobre otros repositorios.
- Elige una fecha de caducidad que cubra el tiempo durante el que quieras usar el regalo. Cuando caduque tendrás que renovarlo y reemplazar el secreto del servicio.
- Copia el token para el paso siguiente. No lo pegues en GitHub, `config.js`, un archivo del proyecto ni un mensaje público.

Si la rama de datos tiene una regla que exige pull requests, el guardado directo será rechazado. Utiliza una rama de datos donde tu token pueda crear commits. No cambies protecciones de otros proyectos.

## 3. Instala el servicio de acceso una sola vez

Necesitas Node.js 22.13 o superior y una cuenta de Cloudflare. Desde una terminal en la carpeta descomprimida de la web:

```powershell
npm install
npx wrangler login
```

Edita `backend/wrangler.jsonc`:

- `GITHUB_OWNER`: tu usuario de GitHub.
- `GITHUB_REPO`: `adviento-lucia-datos`, o el nombre real de tu repositorio **privado**.
- `GITHUB_BRANCH`: normalmente `main`.
- `ALLOWED_ORIGIN`: `https://TU_USUARIO.github.io`, **sin** `/adviento-lucia/` y sin barra final. Si usas dominio propio, introduce su origen HTTPS.
- `ADMIN_USERNAME`: ya es `Manuel`.

Desde la misma carpeta ejecuta estos tres comandos, uno por uno. Cada uno pedirá el valor en privado:

```powershell
npx wrangler secret put ADMIN_PASSWORD --config backend/wrangler.jsonc
npx wrangler secret put GITHUB_TOKEN --config backend/wrangler.jsonc
npx wrangler secret put SESSION_SECRET --config backend/wrangler.jsonc
```

- Para `ADMIN_PASSWORD`, introduce la contraseña elegida para Manuel. Admite la contraseña de cuatro cifras que has solicitado; una más larga protegerá mejor las fotos y la edición. No se incluye ninguna contraseña predeterminada en el repositorio.
- Para `GITHUB_TOKEN`, pega el token del paso 2.
- Para `SESSION_SECRET`, usa un valor aleatorio de al menos 32 caracteres. Puedes generar uno ejecutando `node -e "console.log(require('node:crypto').randomBytes(32).toString('hex'))"` y pegarlo cuando se solicite. No lo guardes en el código.

Si Wrangler pregunta si debe crear el servicio porque todavía no existe, acepta la creación de `adviento-lucia-editor`. Después publica el código:

```powershell
npx wrangler deploy --config backend/wrangler.jsonc
```

Copia la URL que devuelve, por ejemplo `https://adviento-lucia-editor.TU_CUENTA.workers.dev`.

El servicio usa la limitación de intentos de Workers y secretos de entorno; no requiere D1, R2 ni almacenar fotos fuera de GitHub. Consulta los límites y condiciones vigentes de tus cuentas de GitHub y Cloudflare.

## 4. Conecta la página

En el repositorio de la web, edita `docs/config.js` y también `public/config.js` para conservar el ajuste en futuras compilaciones:

```js
window.CALENDAR_CONFIG = {
  apiBase: "https://adviento-lucia-editor.TU_CUENTA.workers.dev"
};
```

Sustituye el ejemplo por la URL real. Guarda los cambios y espera a que termine la publicación de Pages. Esta dirección es pública; no es una contraseña.

## 5. Tus dos enlaces

- **Lucía:** `https://TU_USUARIO.github.io/adviento-lucia/`
- **Manuel:** `https://TU_USUARIO.github.io/adviento-lucia/admin.html`

Entra como **Manuel** con la contraseña que configuraste. Podrás:

1. Abrir cualquiera de los 24 sobres, incluso antes de diciembre, con **Ver todos los días**.
2. Pulsar el lápiz del día para modificar el título, el vale, la frase, la foto y el efecto.
3. Guardar los cambios en el repositorio privado desde el móvil o cualquier ordenador.
4. Pulsar **Ver como Lucía**: abre una pestaña sin privilegios, aunque hayas iniciado sesión como Manuel.

La sesión de edición dura como máximo 8 horas y se conserva solo en esa pestaña. Al cerrar sesión se borra del navegador. Cambiar `ADMIN_PASSWORD` o `SESSION_SECRET` invalida todas las sesiones emitidas anteriormente; vuelve a desplegar el servicio después de cambiar secretos si tu flujo de Cloudflare lo requiere.

## Fechas y privacidad

El año inicial es **2026**. Se cambia en el campo `year` de `calendar.json` del repositorio privado.

El día 1 se abre el 1 de diciembre a las **00:00, hora de Europe/Madrid**; el 2 el día 2, etc. Los días pasados siguen disponibles. Tras el 24 se puede seguir consultando todo el regalo.

La comprobación ocurre en el servidor. Cambiar la fecha del móvil, añadir `preview=true` a una dirección o iniciar sesión en otra pestaña no adelanta las sorpresas de la vista de Lucía. El servicio no entrega los títulos, textos ni fotos futuros a la vista pública. Solo tu sesión de Manuel puede pedirlos antes de tiempo.

Lucía no necesita usuario. **Cualquier persona que tenga la URL de la web podrá ver las sorpresas ya desbloqueadas.** El enlace no es una contraseña. No publiques fotos que no quieras compartir con quienes reciban ese enlace.

## Fotos, textos y guardado

- Se aceptan JPG, PNG y WebP de hasta 20 MB. Para fotos HEIC del móvil, exporta antes como JPG.
- El navegador convierte la foto a JPG, reduce su tamaño a un máximo de 1600 píxeles de lado y a menos de 700 KB. La conversión también evita copiar los metadatos del original.
- Los textos están en `calendar.json`; las fotos, en `photos/day-1.jpg` a `photos/day-24.jpg`, todo en el repositorio privado.
- Foto y texto se guardan juntos en **un único commit**. No tienes que subir archivos a mano ni reconstruir GitHub Pages para actualizar un vale.
- Lucía recibe los cambios al recargar; la página también consulta novedades cada minuto mientras está visible.
- Si editas simultáneamente desde dos dispositivos, el segundo guardado obsoleto se rechaza para no borrar cambios recientes. Cierra el sobre, pulsa **Actualizar desde GitHub** y repite la edición.
- Los sobres abiertos y los vales marcados como disfrutados se recuerdan en el navegador de Lucía, no entre dispositivos.
- Git conserva versiones anteriores. Reemplazar una foto no la borra del historial. Mantén privado el repositorio de datos.

## Si algo falla

- **El calendario se está preparando:** falta la URL real en `docs/config.js`, o estás viendo una versión antigua. Revisa la publicación de Pages y recarga.
- **Failed to fetch / error de red:** comprueba que el Worker está publicado y que `ALLOWED_ORIGIN` coincide exactamente con el origen de la web.
- **Repositorio no encontrado:** revisa propietario, nombre, rama, token y que `calendar.json` esté en la raíz del repositorio privado.
- **El repositorio debe ser privado:** cambia la visibilidad del repositorio de datos a Private; no uses el repositorio público de la web para las fotos.
- **Usuario o contraseña incorrectos:** el usuario es `Manuel` y la contraseña debe coincidir con `ADMIN_PASSWORD`.
- **Demasiados intentos:** espera un minuto. Hay un límite de 5 intentos por minuto e IP, aplicado por la infraestructura de Cloudflare; no sustituye una contraseña larga.
- **Token de GitHub sin permisos:** comprueba que no ha caducado y que tiene Contents Read and write únicamente en el repositorio correcto.
- **Todos los sobres están cerrados para Lucía:** es lo previsto antes del 1 de diciembre del año configurado.

## Cambiar el diseño en el futuro

Los archivos editables están en `src/`. Después de cambiar el diseño:

```powershell
npm install
npm run build
npm test
```

Sube los cambios y la carpeta `docs` regenerada al repositorio de la web. Conserva el `apiBase` real en `public/config.js`, ya que al compilar se copia a `docs/config.js`. Para cambiar fotos y vales no necesitas este proceso: utiliza el panel de Manuel.

## Verificación incluida

Se comprueban los límites de medianoche en Madrid, el bloqueo de contenido futuro, el acceso de Manuel, la firma y alteración de sesiones, los orígenes permitidos, la limitación de intentos, el guardado conjunto en Git, los conflictos entre dispositivos y el rechazo de un repositorio de datos público. Las llamadas a GitHub se simulan en las pruebas: no se ha publicado nada ni se han creado repositorios o cuentas.

## Referencias oficiales

- [Configurar GitHub Pages desde una rama y /docs](https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site)
- [Árboles y commits de GitHub](https://docs.github.com/en/rest/git/trees?apiVersion=2022-11-28)
- [Secretos de Cloudflare Workers](https://developers.cloudflare.com/workers/configuration/secrets/)
- [Límite de intentos de Workers](https://developers.cloudflare.com/workers/runtime-apis/bindings/rate-limit/)
