# ADR 0001: Adopción de Vitest como Test Runner en Backend por compatibilidad ESM

- **Fecha:** 2026-10-30
- **Estado:** Aceptado
- **Módulos afectados:** `backend/`

---

## 1. Contexto

El backend de Nexandría está desarrollado sobre **NestJS** (v11), que por defecto genera un andamiaje de pruebas unitarias y E2E basado en **Jest** bajo un entorno CommonJS (`require()`).

Al integrar el módulo de autenticación con Better Auth y las librerías:
- `@thallesp/nestjs-better-auth` (v2.8.0)
- `@nestjs/config` (v12.x)
- `better-auth` (v1.7.x)

surgió una incompatibilidad en la ejecución de tests. Estos paquetes están empaquetados exclusivamente como **ES Modules puros (ESM, `"type": "module"`)**. Al ejecutar `npm run test` con Jest + `ts-jest`, el loader de Jest fallaba con:

```text
Error [ERR_REQUIRE_ESM]: Must use import to load ES Module: .../@thallesp/nestjs-better-auth/dist/index.mjs
The file contains ESM syntax (import/export) that could not be executed as CommonJS.
```

## 2. Alternativas Evaluadas

1. **Configurar Babel / `@swc/jest` / `transformIgnorePatterns` en Jest:**
   - *Pros:* Mantenía Jest como CLI de tests.
   - *Contras:* Requería transformaciones complejas de dependencias transitivas dentro de `node_modules`, agregando lentitud y fragilidad ante actualizaciones de dependencias en NestJS y Node.js.
2. **Mockear `@thallesp/nestjs-better-auth` en todos los tests unitarios y prescindir de tests E2E con Better Auth:**
   - *Pros:* Evitaba que Jest intentara cargar el módulo ESM en tests aislados.
   - *Contras:* Imposibilitaba pruebas de integración y tests E2E reales (`test/app.e2e-spec.ts`, `test/users-rbac.e2e-spec.ts`), perdiendo cobertura sobre los guards reales de la aplicación.
3. **Adoptar Vitest en el Backend (`vitest.config.mts`):**
   - *Pros:*
     - Soporta módulos ESM nativamente y en paralelo con CommonJS sin configuraciones complejas ni transformaciones manuales en `node_modules`.
     - Es 100% compatible con la sintaxis de NestJS Testing Library (`@nestjs/testing`, `Test.createTestingModule`, mocks de Jest/Vitest).
     - Unifica el test runner del proyecto, ya que el `frontend/` también utiliza Vitest v5.
     - Mayor velocidad de ejecución.
   - *Contras:*
     - Difiere del generador inicial por defecto de `nest new`, requiriendo documentar la decisión para nuevos colaboradores.

## 3. Decisión

Se decide adoptar **Vitest** como el test runner oficial para las pruebas unitarias y E2E de `backend/`:

1. Se creó el archivo de configuración `backend/vitest.config.mts`.
2. Se actualizaron los scripts en `backend/package.json`:
   - `test`: `"vitest run"`
   - `test:watch`: `"vitest"`
   - `test:cov`: `"vitest run --coverage"`
   - `test:e2e`: `"vitest run test/"`
3. Se verifica que **no afecta en absoluto el runtime ni el despliegue de NestJS**: `nest build`, `nest start`, `main.ts`, Docker y producción continúan funcionando de forma idéntica con NestJS y Express.

## 4. Consecuencias

- **Positivas:**
  - Ejecución inmediata y exitosa del 100% de los tests unitarios y E2E en backend sin errores de resolución ESM.
  - Consistencia de herramientas de testing entre frontend y backend (ambos con Vitest).
- **Negativas / Cuidados:**
  - Nuevos tests en backend deben importar sus utilidades o utilizar las variables globales provistas por Vitest (`describe`, `it`, `expect`, `vi`).
