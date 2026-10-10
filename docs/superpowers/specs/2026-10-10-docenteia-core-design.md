# Spec de Diseño: DocenteIA — Plataforma SaaS de IA para Docentes de IESTP

## 1. Resumen Ejecutivo
Construir una plataforma SaaS multi-tenant y white-label completa para institutos de educación superior tecnológica de Perú (IESTP), que permite a docentes, coordinadores y directivos:
1. Ingestar y estructurar planes de estudio y documentos curriculares oficiales según la normativa peruana (RVM 049-2022-MINEDU).
2. Cargar plantillas institucionales en formato DOCX preservando encabezados, membretes y tablas.
3. Generar con IA sílabos completos por unidad didáctica y sesiones de aprendizaje (individuales o en lotes de 16 semanas) con validación pedagógica determinística.
4. Editar, versionar, revisar y exportar en DOCX editable y PDF.
5. Gestionar todo el negocio desde un panel Superadmin global aislado con control de planes comerciales, cuotas, proveedores de IA y ledger inmutable de consumo de tokens.

---

## 2. Arquitectura de la Solución

### 2.1 Stack Tecnológico
- **Frontend & Fullstack:** Next.js 15+ (App Router), React 19, TypeScript estricto.
- **UI & Diseño:** Tailwind CSS, Radix UI, Lucide Icons, diseño accesible WCAG 2.2 AA con temas dinámicos white-label (variables CSS inyectadas en SSR).
- **Base de Datos:** MySQL 8.0 local (`127.0.0.1:3306`, base de datos `docenteia`), charset `utf8mb4`.
- **ORM:** Prisma ORM con migraciones estructuradas.
- **Autenticación & Autorización:** Sesiones cifradas HttpOnly con verificación de tenant y RBAC granular (Superadmin, Tenant Admin, Coordinador, Docente, Revisor).
- **Colas y Tareas Asíncronas:** BullMQ + Redis (con adaptador local de ejecución directa en desarrollo si Redis no está activo).
- **Procesamiento de Documentos:** `docxtemplater`, `pizzip` para DOCX; LibreOffice CLI para conversión a PDF de alta fidelidad.
- **Capa de IA:** Vercel AI SDK Core (`ai`) con adaptador multi-proveedor (OpenAI, Gemini, Groq, DeepSeek, Ollama local) y outputs tipados en Zod.

### 2.2 Estructura del Proyecto
```text
/docs/                      # Especificación técnica, arquitectura, DB y seguridad
/prisma/                    # Schema de Prisma, migraciones y seeders
/public/                    # Assets estáticos y logos predeterminados
/src/
  /app/                     # Next.js App Router (Layouts, Páginas, API REST)
    /(auth)/                # Login multi-tenant, invitaciones
    /(platform)/            # Superadmin de plataforma global
    /(tenant)/              # Paneles de gestión institucional y docente
    /api/v1/                # Rutas de API REST seguras
  /components/              # UI components, layouts, wizards, tablas interactivas
  /lib/                     # Cliente Prisma, utilidades criptográficas, formato
  /server/                  # Capas de dominio
    /academic/              # Lógica de planes curriculares, unidades, créditos, horas
    /ai/                    # Registry de modelos, prompts versionados, RAG
    /billing/               # Transacciones de cuotas, reservas y ledger
    /documents/             # Generador DOCX, render PDF, parseo
    /security/              # Middleware de tenant, RBAC, hash de contraseñas
    /templates/             # Mapeo semántico de campos institucionales
  /worker/                  # Procesamiento de colas y tareas pesadas
```

---

## 3. Fases de Implementación del Plan de Trabajo

- **Fase 1: Fundación y Base del Proyecto**
  - Inicialización del proyecto Next.js + TS + Tailwind.
  - Configuración de Prisma con MySQL (`docenteia`).
  - Modelado completo del esquema de base de datos (`schema.prisma`) y migración inicial.
  - Seed inicial con Superadmin, planes comerciales, proveedores de IA y dos institutos de prueba.
  - Sistema de autenticación con sesiones seguras, middleware de tenancy y resolución de host.

- **Fase 2: Paneles Administrativos y White-Label**
  - Panel Superadmin global (`/(platform)`): Gestión de institutos, planes comerciales, cuotas, proveedores de IA y auditoría.
  - Motor de White-Label: Editor de colores, logos, banners y favicon por instituto con inyección en SSR de CSS variables.
  - Panel Institucional (`/(tenant)`): Dashboard del instituto, gestión de usuarios/docentes, asignación de programas.

- **Fase 3: Módulo Académico Curricular**
  - CRUD de Programas de Estudio, Planes Curriculares y Unidades Didácticas.
  - Carga y extracción de planes de estudios (PDF/DOCX/XLSX).
  - Normalización y pantalla de validación/corrección de datos curriculares (capacidades, horas teóricas/prácticas, créditos).

- **Fase 4: Motor de Plantillas Institucionales OOXML**
  - Carga y versionado de plantillas DOCX para sílabos y sesiones.
  - Asistente de mapeo semántico de placeholders y bloques repetibles (`{{syllabus.weeks}}`, etc.).
  - Validador determinístico de estructura de plantilla y generación de reporte de compatibilidad.

- **Fase 5: Orquestador de IA y Generador de Sílabos**
  - Registry de proveedores de IA con selección por política (`BALANCED`, `ECONOMICAL`, `ADVANCED`).
  - Motor de reservas atómicas y conciliación en `quota_ledger`.
  - Asistente de generación de sílabo en 6 pasos.
  - Editor estructurado de sílabo con regeneración parcial y exportación en DOCX editable y PDF.

- **Fase 6: Generador de Sesiones de Aprendizaje**
  - Generación de sesión individual y masiva (16 semanas del sílabo).
  - Validación de coherencia: suma exacta de minutos en fases didácticas (Inicio, Desarrollo, Cierre).
  - Exportación individual y por lote (ZIP de archivos DOCX y PDF).

---

## 4. Criterios de Aceptación y Pruebas
1. Dos institutos independientes con diferente branding y usuarios funcionando sin fugas de datos.
2. Superadmin capaz de limitar y auditar consumo de tokens en tiempo real con transacciones atómicas.
3. Sílabo y sesiones de aprendizaje generados respetan fielmente las horas y capacidades del plan curricular.
4. Descarga de archivos DOCX editable manteniendo la estructura de la plantilla cargada.
5. Sistema compilando limpiamente con TypeScript estricto y pruebas automatizadas passing.
