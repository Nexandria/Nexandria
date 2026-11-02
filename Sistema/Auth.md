# Auth

Última revisión: 2026-10-30. Esto documenta cómo funciona el login hoy en el código, no cómo "debería" funcionar. Fuentes: `backend/src/auth/auth.ts`, `backend/src/mail/mailer.ts`, `backend/prisma/schema.prisma`, `frontend/src/auth/client.ts`, `frontend/src/lib/dev/`, `frontend/src/auth/routes/RoleHome.tsx`, `frontend/src/auth/pages/{Login,Register,Recovery}.tsx`.

## Qué es better-auth y dónde vive

Es una librería que maneja todo el ciclo de login/signup/sesión por nosotros: guarda contraseñas hasheadas, genera cookies de sesión, habla con Google/Facebook para el login social, etc. Vive del lado del backend (`backend/src/auth/auth.ts` es su configuración completa) y el frontend le habla a través de un cliente (`frontend/src/auth/client.ts`).

Las rutas de better-auth se montan automáticamente bajo `/api/auth` en el backend (es el valor por defecto de la librería, no lo cambiamos). Eso importa para la configuración de OAuth, ver abajo.

## Las 3 pantallas

### Signup (`/register`)
Pide: email, nombre de usuario (username), contraseña y repetir contraseña. También hay botones de "Continuar con Google/Facebook".

Al mandar el formulario, llamamos `signUp.email({ email, password, username, name: username })`. Better-auth crea el usuario y, apenas termina, manda automáticamente un código de verificación (OTP) al mail — eso lo hace el plugin `emailOTP` con la opción `sendVerificationOnSignUp: true`, no lo disparamos nosotros a mano. El frontend muestra una segunda pantalla para que el usuario ponga ese código, y lo confirma llamando `authClient.emailOtp.verifyEmail({ email, otp })`.

### Login (`/login`)
Pide un solo campo que puede ser email o username, más la contraseña (y los mismos botones de OAuth). En el frontend decidimos cuál endpoint llamar mirando si lo que tipeó tiene un `@`:
- Si parece email → `signIn.email({ email, password })`.
- Si no → `authClient.signIn.username({ username, password })` (esto lo agrega el plugin `username`).

### Recovery (`/recovery`)
Pide solo el email. No usamos el flujo clásico de "resetear contraseña con un link que vence" — usamos **magic link**: le mandamos un link al mail y, al clickearlo, el usuario queda logueado directo (no necesita recordar ni cambiar la contraseña vieja). Lo dispara `authClient.signIn.magicLink({ email, callbackURL: '/app' })`.

Un detalle importante: configuramos el plugin con `disableSignUp: true`. Sin eso, mandar un magic link a un email que no tiene cuenta crearía una cuenta nueva ahí mismo — lo cual se saltearía el signup normal (que exige username). Con `disableSignUp: true`, si el email no existe, simplemente no pasa nada (y el frontend no te dice si existe o no, para no filtrar esa información).

## OAuth: Google y Facebook

### Qué datos llegan
Google y Facebook, cuando el usuario autoriza, solo nos dan: nombre, email, foto y si el email está verificado. **No** nos dan un username — ese concepto no existe en esos providers.

Como nuestro modelo exige username, cuando alguien entra por primera vez vía OAuth se lo generamos nosotros: tomamos su nombre, lo "normalizamos" (sin tildes, sin espacios, todo minúscula) y le pegamos un número al final, probando `nombre1`, `nombre2`, etc. hasta encontrar uno libre. Esto pasa en un hook (`databaseHooks.user.create.before`) en `auth.ts`, y corre justo antes de guardar el usuario nuevo.

### Rutas de callback para configurar en cada provider

Better-auth expone el callback en `/api/auth/callback/<provider>` bajo el `BETTER_AUTH_URL` configurado. Con los valores de ejemplo del repo:

- **Google** (Google Cloud Console → Credenciales → OAuth Client → "Authorized redirect URIs"):
  `http://localhost:3000/api/auth/callback/google` (dev) — en producción, reemplazá el host por el dominio real del backend.
- **Facebook** (Meta for Developers → tu app → Facebook Login → Settings → "Valid OAuth Redirect URIs"):
  `http://localhost:3000/api/auth/callback/facebook` (dev) — mismo reemplazo en producción.

El puerto/host sale de `BETTER_AUTH_URL` (`backend/.env`). Si ese valor cambia, estas URLs cambian con él.

Las credenciales (`GOOGLE_CLIENT_ID`/`SECRET`, `FACEBOOK_CLIENT_ID`/`SECRET`) van en `backend/.env`. Si están vacías, ese provider queda deshabilitado solo (no rompe nada, simplemente no aparece como opción real).

## El modelo `User` (lo que importa de auth)

Campos relevantes hoy: `email`, `username` (único, puede ser `null` momentáneamente si alguien entra por OAuth antes de que el hook le asigne uno — en la práctica nunca queda `null` porque el hook corre siempre), `name`, `role` (string libre, sin relación con better-auth — nace en `"user"` por default de la base, `@default("user")` en el schema; nadie lo cambia automáticamente después, eso se asigna a mano), y `onboard`.

### `onboard`
Un campo de texto con 3 valores posibles: `none`, `required`, `verified`. Hoy **todo usuario nuevo nace en `none`** (es el default en la base) y **no hay ninguna otra lógica que lo cambie todavía** — ni pantalla de onboarding, ni gate que bloquee funciones según el nivel. Es a propósito: se deja el campo listo para cuando se construya ese flujo, sin construir el flujo en sí ahora.

### Lo que sacamos (y por qué)
- **`sedeId`** y la relación a `Sede`: no estaba en el alcance de lo que se estaba definiendo para auth, se sacó del modelo `User`. El modelo `Sede` en sí sigue existiendo (es parte del dominio de bibliotecas, no de auth), simplemente ya no tiene una relación directa a `User`.
- **El plugin `admin`** de better-auth: se sacó antes (ver historial de este documento). La autorización por rol sigue funcionando porque el guard `@Roles()` lee `session.user.role` directo, sin depender de ningún plugin.

## Qué le llega al backend en cada request

Cada request (salvo las marcadas `@AllowAnonymous()`) pasa por un guard que le pregunta a better-auth "¿quién es este?". Esa respuesta (`session.user`) ya trae **todos** los campos del usuario — `role`, `username`, `onboard`, todo — en el mismo viaje. No hace falta hacer una consulta aparte a la base para leer el rol o el nivel de onboarding: ya está ahí.

Activamos `session.cookieCache` (en `auth.ts`) para que esa consulta **no** le pegue a la base de datos en cada request: la respuesta queda guardada 60 segundos en una cookie firmada, y mientras no venza, better-auth la usa directo sin tocar Postgres.

El trade-off real de esto: si algo le cambia el `role` o el `onboard` a un usuario que tiene una sesión activa, ese usuario puede seguir viendo el valor viejo hasta por 60 segundos (hasta que la cookie vence y se refresca desde la base). Hoy no es grave porque nada cambia esos campos automáticamente todavía — pero es la razón por la que, si en el futuro se banea a alguien o se le saca un rol en caliente, puede tardar hasta un minuto en hacerse efectivo salvo que se invalide la cookie a mano.

## Mail: OTP de verificación y magic link

Better-auth no manda mails por su cuenta — solo te avisa "mandá este código" o "mandá este link" y la app tiene que efectivamente enviarlo. Eso lo hace `backend/src/mail/mailer.ts`, un wrapper chico sobre `nodemailer` que lee la configuración SMTP de las env vars (`SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `SMTP_FROM`, en `backend/.env`). Mientras esas variables estén vacías, los mails no se van a enviar (van a fallar en silencio o con error de conexión, según el proveedor) — hay que configurar un SMTP real (Mailtrap para probar, o el proveedor que se use en producción) antes de que el signup y el recovery funcionen de punta a punta.

## A dónde va cada rol: `/app`

`/app` no es una pantalla: es el destino genérico después de loguearse, y también adonde `ProtectedRoute` manda a quien no tiene el rol requerido. `RoleHome` (`frontend/src/auth/routes/RoleHome.tsx`) lo resuelve según el rol:

| Rol | Destino |
| --- | --- |
| `user`, `student` | `/inicio` |
| `netAdmin` | `/admin` |
| `branchAdmin`, `librarian`, sin rol | Se queda en `/app`, con un aviso de que todavía no hay pantalla |

Los roles sin pantalla no redirigen a ningún lado a propósito: `ProtectedRoute` los devolvería a `/app` y quedarían en loop. Cuando un rol tenga pantalla propia, se agrega al mapa `HOME_BY_ROLE` de ese archivo.

## Modo dev: simular el rol sin backend

Sirve para ver y desarrollar las pantallas de cada rol sin levantar el backend ni tener un usuario por rol. Es solo para desarrollo de frontend: **no sirve para probar permisos reales**, porque el backend no participa.

### Cómo se usa
1. Crear `frontend/.env.local` (está en `.gitignore`) con `VITE_MOCK_AUTH=true`.
2. Reiniciar `npm run dev`.
3. Elegir el rol en el selector **Rol dev** (`DevRoleSwitcher.tsx`), abajo a la derecha. Están los 5 roles y "sin sesión", que hace falta para ver `/login` y `/register`. El cambio se aplica al instante y queda guardado en `localStorage` (`nexandria:dev-role`). La primera vez arranca como `user`.

Para volver al login real, borrar la línea o ponerla en `false` y reiniciar.

### Qué se simula
Todo está en `frontend/src/lib/dev/`:
- **`useSession` y `signOut` (`session.ts`):** `auth/client.ts` los reemplaza por sus versiones mock, así que las rutas y pantallas no se enteran. Cerrar sesión deja el selector en "sin sesión".
- **`GET /users` y `GET /users/me` (`mock-api.ts`):** un interceptor de axios, registrado en `lib/api.ts`, responde con datos fijos y respeta el rol simulado igual que `@Roles()` en el backend: 200, 403 o 401 según corresponda. Sin esto darían siempre 401, porque no hay cookie real.

### Qué sigue siendo real
- **Cualquier otro endpoint va al backend.** La búsqueda (`/search/books`) necesita el backend levantado.
- **Los formularios de login, registro, OTP y recovery y los botones de OAuth llaman al backend.** Se puede ver el diseño, pero no completar el flujo sin backend.

Si una pantalla nueva usa un endpoint protegido, hay que agregarlo en `mockResponse` (`mock-api.ts`) para que funcione en este modo.

### Por qué no llega a producción
`MOCK_AUTH` exige `import.meta.env.DEV`, que el build reemplaza por `false`. Vite elimina el código del mock del bundle aunque la variable esté definida (verificado buscando el mock en un build con `VITE_MOCK_AUTH=true`, el 2026-10-30). Además, `MODE !== 'test'` lo mantiene apagado en los tests aunque exista `.env.local`.

Aunque alguien forzara un rol en el navegador, no ganaría ningún acceso: el backend valida cada request con la sesión real.
