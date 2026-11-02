# Suite de Tests: Módulo de Autenticación (Auth)

Diseño y especificación de pruebas para el módulo de autenticación de Nexandría, abarcando las capas de **Frontend** (Vitest + Testing Library) y **Backend** (NestJS + Jest + Supertest).

---

## 1. Frontend (Vitest + Testing Library)

El objetivo en el frontend es validar la lógica de validación de entradas, la interacción del usuario con los formularios de autenticación y la protección de rutas según el estado de la sesión y el rol.

### 1.1. Esquemas y Utilidades (`src/auth/validation.test.ts`)
Pruebas unitarias sobre los esquemas de validación Zod:
- **`loginSchema`**:
  - Acepta credenciales con correo o nombre de usuario no vacío.
  - Rechaza campos vacíos con los mensajes de error correspondientes.
- **`registerSchema`**:
  - Exige contraseña de mínimo 8 caracteres.
  - Exige que el nombre de usuario cumpla con longitud (3 a 24 caracteres) y caracteres permitidos (alfanumérico y guión bajo).
  - Rechaza el formulario si `password` y `repeatPassword` no coinciden (`"Las contraseñas no coinciden"`).
- **`otpSchema`**:
  - Valida que el código sea de exactamente 6 dígitos numéricos.
- **`firstIssueMessage`**:
  - Retorna el primer mensaje amigable ante fallos de validación de Zod.

### 1.2. Formulario de Inicio de Sesión (`src/auth/forms/LoginForm.test.tsx`)
Pruebas de componente para el inicio de sesión:
- **Renderizado inicial**: Labels accesibles para identificador, contraseña y botón de submit ("Entrar").
- **Interacción de usuario**: Simulación de escritura fluida en los inputs con `userEvent`.
- **Detección de método (Email vs Username)**:
  - Si el identificador incluye `@`, invoca `signIn.email({ email, password })`.
  - Si el identificador no incluye `@`, invoca `authClient.signIn.username({ username, password })`.
- **Manejo de errores**: Si la API retorna un error de autenticación, renderiza el mensaje accesible con `role="alert"`.
- **Redirección**: Al autenticarse correctamente, ejecuta la navegación hacia `/app`.

### 1.3. Formulario de Registro (`src/auth/forms/RegisterForm.test.tsx`)
Pruebas de componente para la creación de cuenta:
- **Renderizado**: Inputs de email, username, contraseña y confirmación de contraseña.
- **Validación del lado del cliente**:
  - Si las contraseñas son distintas, interrumpe el submit antes de consultar a la API y muestra la alerta correspondiente.
- **Registro exitoso**:
  - Invoca `signUp.email({ email, username, password, name: username })`.
  - Dispara el callback `onRegistered(email)` para hacer la transición a la pantalla de verificación OTP.
- **Error del servidor**: Muestra mensajes devueltos por el backend (por ejemplo, si el correo o usuario ya están en uso).

### 1.4. Formulario de Verificación OTP (`src/auth/forms/VerifyOtpForm.test.tsx`)
Pruebas de componente para la verificación del código de 6 dígitos:
- **Validación de longitud**: Rechaza códigos incompletos o con más de 6 caracteres.
- **Verificación exitosa**: Invoca `authClient.emailOtp.verifyEmail({ email, otp })` y ejecuta `onVerified()`.
- **Código incorrecto o expirado**: Muestra el mensaje de rechazo de la API sin recargar la página.

### 1.5. Formulario de Recuperación (`src/auth/forms/RecoveryForm.test.tsx`)
Pruebas sobre el flujo de Magic Link:
- **Validación de correo**: Exige un formato de email válido.
- **Envío de magic link**: Invoca `authClient.signIn.magicLink({ email, callbackURL: '/app' })`.
- **Feedback**: Renderiza el mensaje confirmando el envío del enlace a la casilla del usuario.

### 1.6. Protección de Rutas (`src/auth/routes/ProtectedRoute.test.tsx`)
Pruebas de integración sobre el enrutador y control de acceso:
- **Estado de carga**: Mientras `isPending` es verdadero, muestra el indicador de carga.
- **Usuario anónimo**: Si no hay sesión activa, redirige a `/login` guardando la ubicación previa en `state.from`.
- **Acceso por rol denegado**: Si la ruta requiere roles específicos (ej. `['netAdmin']`) y el usuario logueado tiene rol `user` o `student`, redirige a `/app`.
- **Acceso permitido**: Si el usuario cumple con el rol requerido, renderiza la vista solicitada (`<Outlet />`).

### 1.7. Destino por Rol (`src/auth/routes/RoleHome.test.tsx`)
- **Roles con pantalla**: `user` y `student` van a `/inicio`; `netAdmin` va a `/admin`.
- **Roles sin pantalla**: `branchAdmin`, `librarian` y los usuarios sin rol ven el aviso, sin redirigir (evita el loop con `ProtectedRoute`).

### 1.8. Modo Dev (`src/lib/dev/session.test.ts`, `src/lib/dev/mock-api.test.ts`)
- **Apagado en tests**: `MOCK_AUTH` es falso.
- **Rol guardado**: Un valor ausente o inválido en `localStorage` cae en `user`.
- **Sesión mock**: Cambiar de rol actualiza la sesión; `signOut` la deja vacía.
- **API mock**: `/users` responde 200, 403 o 401 según el rol; `/users/me` responde 200 con sesión y 401 sin ella; el resto de los endpoints no se intercepta.

---

## 2. Backend (NestJS + Jest + Supertest)

El objetivo en el backend es verificar la robustez de los hooks de Better Auth, el servicio de envío de correos, y el ciclo completo de autenticación y autorización (RBAC) mediante tests End-to-End.

### 2.1. Tests Unitarios de Lógica Interna
- **Normalización y generación de usuarios (`src/auth/auth-hooks.spec.ts`)**:
  - Función `slugify`: Comprobar que remueva acentos (`José` $\rightarrow$ `jose`), caracteres especiales y recorte el tamaño a 24 caracteres.
  - Función `generateUsername`: Comprobar que genere sufijos numéricos secuenciales en caso de colisión con usuarios ya existentes en la base de datos (flujo OAuth).
- **Servicio de correo (`src/mail/mailer.spec.ts`)**:
  - Verificar que `sendMail` prepare y despache el mensaje usando el transporte configurado de Nodemailer.

### 2.2. Tests End-to-End de Autenticación (`test/auth.e2e-spec.ts`)
Pruebas sobre los endpoints expuestos por Better Auth bajo `/api/auth`:
- **Alta de usuario y disparo de OTP**:
  - `POST /api/auth/sign-up/email`: Crea el registro en PostgreSQL con `emailVerified = false` y dispara automáticamente el envío del código OTP de 6 dígitos.
- **Verificación de OTP**:
  - `POST /api/auth/email-otp/verify-email`:
    - Con código inválido: Rechaza con error.
    - Con código correcto: Marca `emailVerified = true` y genera la sesión.
- **Inicio de sesión**:
  - `POST /api/auth/sign-in/email`: Con credenciales correctas retorna la sesión y establece la cookie de sesión (`better-auth.session_token`).
  - `POST /api/auth/sign-in/username`: Mismo comportamiento identificándose por username.
  - Credenciales incorrectas: Rechazo con código `401 Unauthorized`.
- **Cierre de sesión**:
  - `POST /api/auth/sign-out`: Invalida la sesión activa y limpia las cookies.

### 2.3. Tests End-to-End de Control de Acceso por Roles (`test/users-rbac.e2e-spec.ts`)
Pruebas sobre los guards y decoradores de roles en NestJS (`@Roles`, `@Session`):
- **Sesión propia (`GET /users/me`)**:
  - Sin cookie de sesión $\rightarrow$ `401 Unauthorized`.
  - Con cookie de sesión $\rightarrow$ `200 OK` retornando los datos de usuario y rol.
- **Listado restringido (`GET /users`)**:
  - Usuario con rol estándar (`user` o `student`) $\rightarrow$ `403 Forbidden`.
  - Usuario con rol de administrador (`netAdmin`) $\rightarrow$ `200 OK` con la información administrativa.

---

## 3. Comandos de Ejecución

```bash
# Frontend (dentro de /frontend)
npm run test         # Modo watch interactivo
npm run test:run     # Ejecución única (CI/CD)

# Backend (dentro de /backend)
npm run test         # Tests unitarios
npm run test:e2e     # Tests End-to-End
```
