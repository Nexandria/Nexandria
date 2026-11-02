Implementado con [Better Auth](https://better-auth.com) (v1.7.12) como base para sesiones, formularios de registro/login, login social y control de roles. Corre en el [[Red|backend]], y el [[Red|frontend]] solo consume su sesión.

# Por qué no el plugin de Organization
Better Auth tiene dos formas de modelar roles: el plugin **organization** (pensado para multi-tenant con permisos finos por recurso, tipo "project: create/update/delete") y el plugin **admin** (un campo `role` simple en el usuario). Se eligió **admin**, porque el modelo real de Nexandría no necesita permisos por recurso dentro de una sede: un Librarian o un BranchAdmin pertenecen *a lo sumo a una* [[Sede]], y el guard que hace falta es "puede entrar a esta pantalla/endpoint" o no, no "puede crear pero no borrar".

# Roles
Definidos en `backend/src/auth/roles.ts` (y espejados en `frontend/src/lib/roles.ts`, hay que mantener ambos sincronizados a mano porque son paquetes separados):

| Rol | Alcance | `sedeId` | Cómo se crea |
|---|---|---|---|
| `netAdmin` | toda la [[Red]] | `null` | a mano (es el único rol "admin de plataforma" en Better Auth: puede banear/impersonar) |
| `branchAdmin` | 1 [[Sede]] | la suya | lo crea el [[Red\|Administrador de Red]] al dar de alta la sede |
| `librarian` | 1 [[Sede]] | la suya | lo crea el [[Sede\|Administrador de Sede]] de esa sede |
| `student` | 1 [[Sede]] escolar | la suya | lo crea BranchAdmin/librarian de esa sede (individual o por CSV, *pendiente*) |
| `user` | toda la [[Red]] | `null` | autorregistro — es el rol con el que nace cualquier cuenta nueva |

Solo `user` se autoasigna (`defaultRole` del plugin admin). El resto de los roles los asigna siempre el backend — ningún usuario puede elegir su propio rol ni su `sedeId`.

# Sesión y formularios
- Email + contraseña: `authClient.signIn.email` / `signUp.email` — el flujo de [[Lector|User]] común.
- Social (Google / Facebook): vía `socialProviders`. Si `GOOGLE_CLIENT_ID`/`FACEBOOK_CLIENT_ID` están vacíos en `.env`, Better Auth no ofrece ese proveedor (no explota, se cae silenciosamente).
- Las cuentas de staff (NetAdmin, BranchAdmin, Librarian) y de [[Alumno|Student]] **no se autorregistran**: nacen desde un panel con contraseña generada. Falta conectar el envío de esa contraseña por correo (ver Pendiente).

# Guards
**Backend** (librería `@thallesp/nestjs-better-auth`):
- Todas las rutas requieren sesión por default (hay un `AuthGuard` global).
- `@AllowAnonymous()` en un controller/endpoint lo hace público (usado hoy en `/` y `/health`).
- `@Roles(['netAdmin'])` restringe por `user.role` exacto — ejemplo real en `src/users/users.controller.ts` (`GET /users` solo para `netAdmin`).
- `@Session() session: UserSession` para leer el usuario autenticado dentro de un endpoint.

**Frontend** (`frontend/src/routes/ProtectedRoute.tsx`):
- Envuelve rutas de React Router. Sin sesión → redirige a `/login`.
- Con `roles={[...]}` además exige que `session.user.role` esté en esa lista; si no, redirige a `/`.
- Esto solo **oculta pantallas**. La autorización real la hace siempre el backend con `@Roles` — nunca hay que confiar en que el frontend la aplicó.

# Modelo de datos (Prisma)
Better Auth generó `User`, `Session`, `Account`, `Verification` corriendo `npx auth generate` (regenerar cada vez que cambie la config de `auth.ts`, después migrar con `prisma migrate dev`). A mano se agregó:
- `Sede` (`id`, `nombre`, `codigo` único, `tipo`: `COMUN` | `ESCOLAR`).
- `User.sedeId` (opcional) + relación a `Sede`.

El cliente de Prisma (`backend/src/prisma/prisma-client.ts`) es un singleton compartido entre Better Auth (`auth.ts`) y el resto de la app (`PrismaService`), para no abrir dos pools de conexión a la vez.

# Pendiente
Lo que todavía no está conectado, en orden de dependencia con otras partes del sistema:
- [[Identidad|Verificación de identidad]] por DNI (PDF417): hoy no hay campo `dni`/`fechaNacimiento`/`identityVerified` en `User` — se agrega cuando se construya ese flujo, no antes, para no adivinar la forma del dato.
- Puntaje, avatar y apellido de [[Lector|User]] — perfil extendido, mismo motivo que arriba.
- Envío por correo de la contraseña generada al crear BranchAdmin/Librarian/[[Alumno|Student]].
- Alta masiva de [[Alumno|Student]] por CSV/Excel.
- No hay pantalla para asignar `sedeId`: hoy se setea directo en base/backend.
