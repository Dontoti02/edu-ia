# Arquitectura del Sistema: DocenteIA (Plataforma SaaS Multi-Tenant)

## 1. Visión General y Topología
**DocenteIA** es una plataforma SaaS multi-tenant diseñada para institutos de educación superior tecnológica (IESTP) de Perú. La arquitectura sigue un modelo de **Monolito Modular con Workers Asíncronos Desacoplados**.

```
+---------------------------------------------------------------------------------+
|                                 Cliente Web                                     |
|           (Superadmin / Administrador de Instituto / Docente / Coordinador)     |
+---------------------------------------------------------------------------------+
                                       |
                                       v
+---------------------------------------------------------------------------------+
|                       Next.js App Router (Node.js 22 LTS)                      |
|                                                                                 |
|  [Capas de Rutas]                                                              |
|   - /(public)   : Landing institucional y login dinámico                        |
|   - /(platform) : Panel Superadmin global (admin.dominio / host reservado)     |
|   - /(tenant)   : Panel del instituto (slug / subdominio / dominio propio)      |
|   - /api/v1     : API REST versionada, DTOs con Zod, OpenAPI                    |
|                                                                                 |
|  [Capa de Aplicación y Servidor (src/server)]                                   |
|   - TenancyContext & Security Resolution                                        |
|   - RBAC / ABAC Matrix & Permisos Granulares                                    |
|   - Domain Services:                                                            |
|       * Academic (Planes, Unidades Didácticas, Capacidades, Indicadores)       |
|       * Templates (Plantillas DOCX, Marcadores Semánticos, Mapeo)               |
|       * AIService (Orquestación, Routing de Modelos, RAG, Structured Output)   |
|       * Billing & Quotas (Ledger inmutable, Reservas atómicas de tokens)       |
|       * Documents (Generación y revisión de Sílabos y Sesiones)                |
+---------------------------------------------------------------------------------+
          |                                      |                     |
          v                                      v                     v
+-----------------------+              +------------------+   +-------------------+
|     MySQL 8.0 LTS     |              |   Redis 7+       |   | Storage Privado   |
|  (Prisma ORM Tipado)  |              |  (BullMQ Colas)  |   | (S3 / MinIO /     |
|  - Multi-tenant real  |              |  - Generaciones  |   |  Local seguro)    |
|  - Ledger inmutable   |              |  - Conversión PDF|   | - Archivos fuente |
|  - UTF8mb4            |              |  - Extracciones  |   | - Plantillas DOCX |
+-----------------------+              +------------------+   +-------------------+
                                                 |
                                                 v
                               +-----------------------------------+
                               |     Workers Autónomos (Node.js)    |
                               |  - Procesamiento pesado BullMQ    |
                               |  - Parser y extracción curricular |
                               |  - Render DOCX (Docxtemplater)    |
                               |  - Conversión PDF (LibreOffice)   |
                               |  - Llamadas a Proveedores IA      |
                               +-----------------------------------+
```

---

## 2. Decisiones Técnicas y Justificación

| Componente | Elección | Justificación |
|---|---|---|
| **Framework** | Next.js 15+ (App Router) + React 19 + TypeScript | Renderizado híbrido SSR/Client, Server Actions, Route Handlers, soporte de tipado integral. |
| **Estilos e Interfaz** | Tailwind CSS + Lucide Icons + Radix UI / shadcn patterns | Diseño visual premium, responsive, accesible (WCAG 2.2 AA) y tokens de temas dinámicos white-label en CSS variables. |
| **Base de Datos** | MySQL 8.0 / 8.4 LTS | Requisito estricto. Tablas relacionales con `tenant_id` obligatorio en todas las entidades institucionales, transacciones ACID para reservas de cuotas. |
| **ORM** | Prisma ORM | Tipado estricto extremo a extremo, migraciones declarativas y consultas seguras contra inyección SQL. |
| **Colas y Asincronía** | Redis + BullMQ (con fallback local/in-process en dev) | Trabajos pesados desacoplados de peticiones HTTP (generación por lotes de 16 sesiones, conversión OOXML a PDF). |
| **Gestor de IA** | Vercel AI SDK Core (`ai`) + adaptadores | Soporte unificado de OpenAI, Anthropic, Google Gemini, Groq, Ollama y APIs compatibles con OpenAI. Structured outputs con validación Zod determinística. |
| **Documentos OOXML** | `docxtemplater` + `pizzip` + motor base | Manipulación fiel de plantillas DOCX sin romper membretes, tablas complejas ni estilos visuales. |
| **Conversión PDF** | Worker con LibreOffice CLI / motor compatible | Generación de vistas previas y entregables finales en PDF de alta fidelidad. |
| **Validación de Datos** | Zod en todas las capas | Validación de entradas en API, salidas de IA y estructuras de plantillas. |

---

## 3. Modelo Multi-Tenant y Aislamiento

1. **Resolución del Tenant:**
   - Detectado en middleware/servidor mediante:
     1. Cabecera Host (Subdominio `instituto.dominio.pe` o dominio propio `docentes.iestp.edu.pe`).
     2. Host reservado para superadmin (`admin.dominio.pe` o `/platform`).
     3. Ruta con prefijo seguro de fallback (`/i/:tenantSlug`).
   - El `tenantId` jamás se toma a ciegas de parámetros cliente sin verificar la pertenencia de la sesión activa.

2. **Aislamiento en Base de Datos:**
   - Toda tabla de datos institucionales contiene `tenant_id VARCHAR(36)` indexado con foreign keys compuestas.
   - Toda consulta en el repositorio de aplicación pasa por un `TenantContext` validado.
   - Claves únicas compuestas: `@@unique([tenantId, code])` o `@@unique([tenantId, slug])` para evitar colisiones entre institutos.

3. **White-Label Dinámico:**
   - Cada tenant almacena en `TenantBranding`: logo, favicon, banner de login, colores primario/secundario/acento, textos de bienvenida y tipografía.
   - En cada carga de página, los tokens CSS (`--institution-primary`, etc.) se inyectan en tiempo de servidor para evitar saltos visuales (FOUC).

---

## 4. Control de Cuotas y Ledger Inmutable

Para evitar el consumo descontrolado o llamadas paralelas fraudulentas:
1. **Pre-reserva atómica:** Antes de ejecutar una solicitud de IA, una transacción MySQL crea un `QuotaReservation` y verifica si el instituto tiene saldo/créditos disponibles en su `QuotaPeriod`.
2. **Generación con IA:** Se envía la llamada al proveedor mediante el worker asíncrono.
3. **Conciliación final:** Al finalizar, se registra en `QuotaLedger` (append-only) los tokens exactos de entrada y salida, el costo del proveedor y se liquida la reserva. Si falla, la reserva se libera sin cobrar créditos comerciales indebidos.

---

## 5. Estructura de Directorios del Código Fuente

```text
/docs/                      # Documentación de arquitectura, base de datos y seguridad
/prisma/                    # Schema de Prisma, migraciones y seeders
/public/                    # Assets estáticos globales y marcas por defecto
/src/
  /app/                     # Next.js App Router
    /(auth)/                # Rutas de login institucional, recuperación y activación
    /(platform)/            # Rutas del Superadmin global
    /(tenant)/              # Rutas del panel institucional y del docente
    /api/v1/                # Controladores de API REST tipada
  /components/              # Componentes de UI (UI Library, Dialogs, Wizards, Layouts)
  /lib/                     # Utilidades compartidas (crypto, formatting, prisma client)
  /server/                  # Capa de dominio y lógica de negocio
    /academic/              # Curriculo, unidades didácticas, capacidades, indicadores
    /ai/                    # Registry de modelos, prompts versionados, RAG
    /billing/               # Cuotas, transacciones, ledger y planes
    /documents/             # Parseo y generación de DOCX / PDF, validadores
    /security/              # RBAC, auth, resolución de tenant, tokens seguros
    /templates/             # Mapeo semántico de plantillas y extracción de marcadores
  /worker/                  # Procesadores de colas BullMQ para tareas en segundo plano
```
