# Modelo y Políticas de Seguridad: DocenteIA SaaS

## 1. Modelo de Amenazas y Aislamiento Multi-Tenant
En un entorno SaaS donde múltiples instituciones educativas coexisten en la misma base de datos MySQL, el aislamiento estricto es la máxima prioridad:

1. **Aislamiento a Nivel de Aplicación:**
   - La resolución de identidad de tenant se realiza exclusivamente en el servidor a través del dominio/subdominio o la sesión autenticada.
   - El cliente **nunca** puede enviar un `tenantId` arbitrario en el cuerpo JSON o query params para alterar el contexto de consulta.
   - Todo servicio de dominio y repositorio exige explícitamente un objeto `TenantContext` inmutable.

2. **Aislamiento en Consultas SQL / ORM:**
   - Todas las consultas a tablas que pertenezcan al ámbito del instituto incluyen un predicado obligatorio `where: { tenantId: ctx.tenantId, ... }`.
   - Se aplican claves foráneas y restricciones únicas compuestas `(tenant_id, code)` para evitar accesos cruzados mediante identificadores numéricos o slugs predecibles.

3. **Aislamiento de Almacenamiento de Archivos:**
   - Los archivos subidos (planes curriculares, plantillas DOCX, sílabos generados) se guardan en rutas particionadas: `tenants/{tenantId}/{category}/{fileId}.ext`.
   - Prohibido el acceso directo o enlaces estáticos públicos en `/public`.
   - Todas las descargas se sirven mediante enlaces firmados con tiempo de expiración corto (ej. 15 minutos) o mediante endpoints que verifican previamente la membresía activa del usuario en el tenant.

---

## 2. Autenticación y Gestión de Sesiones

1. **Almacenamiento de Credenciales:**
   - Contraseñas procesadas mediante función de derivación segura (Argon2id o bcrypt con factor de coste >= 12).
   - Rate limiting defensivo en endpoints de autenticación para mitigar ataques de fuerza bruta y enumeración de usuarios.
2. **Cookies de Sesión:**
   - Cookies `HttpOnly`, `Secure` (en producción), `SameSite=Lax` con expiración rotativa y revocación centralizada en base de datos.
   - Protección contra Cross-Site Request Forgery (CSRF) en todas las operaciones que alteren estado.
3. **MFA Obligatorio para Superadmin:**
   - El acceso al panel global de plataforma exige autenticación multifactor basada en TOTP (RFC 6238).
   - Prohibido el auto-registro abierto para roles de Superadmin; su aprovisionamiento inicial se realiza mediante script de bootstrap CLI seguro.

---

## 3. Matriz de Autorización (RBAC y ABAC)

| Permiso | Superadmin | Admin Tenant | Coordinador | Docente | Revisor |
|---|:---:|:---:|:---:|:---:|:---:|
| `platform.manage` | ✅ | ❌ | ❌ | ❌ | ❌ |
| `tenants.create` | ✅ | ❌ | ❌ | ❌ | ❌ |
| `tenant.branding.update` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `users.invite` | ❌ | ✅ | ❌ | ❌ | ❌ |
| `curricular.plans.approve`| ❌ | ✅ | ✅ | ❌ | ❌ |
| `templates.manage` | ❌ | ✅ | ✅ | ❌ | ❌ |
| `syllabi.generate` | ❌ | ❌ | ✅ | ✅ | ❌ |
| `syllabi.edit` | ❌ | ❌ | ✅ (asignado) | ✅ (propio) | ❌ |
| `documents.approve` | ❌ | ✅ | ✅ | ❌ | ✅ |
| `billing.view` | ✅ (global) | ✅ (propio) | ❌ | ❌ | ❌ |

Regla contextual (ABAC): Un docente con rol `TEACHER` solo puede editar o solicitar generación para las unidades didácticas que le han sido asignadas para el periodo lectivo en curso.

---

## 4. Seguridad en Inteligencia Artificial y LLMs

1. **Defensa contra Prompt Injection:**
   - Los documentos subidos por los usuarios (archivos de plan curricular en PDF/DOCX) se consideran **datos no confiables**.
   - En los prompts del sistema se aíslan los textos extraídos dentro de delimitadores claros (`<source_document>...</source_document>`) y se instruye al modelo a tratar el contenido estrictamente como material de análisis, ignorando cualquier instrucción embebida que intente cambiar el comportamiento del sistema.
2. **Salidas Estructuradas Determinísticas:**
   - La respuesta del LLM debe ajustarse forzosamente al esquema Zod esperado mediante llamadas tipadas de AI SDK.
   - Se validan sumas de horas, coherencia de créditos y estructura jerárquica con reglas de negocio determinísticas antes de persistir cualquier documento.
3. **Protección de Credenciales (BYOK - Bring Your Own Key):**
   - Cuando un instituto suministra su propia API Key, esta se encripta en reposo utilizando **AES-256-GCM** con una clave maestra protegida en variables de entorno.
   - En la interfaz de usuario solo se muestran los últimos 4 caracteres; nunca se registra en logs ni se envía al navegador del cliente.

---

## 5. Procesamiento Seguro de Archivos y OOXML

1. **Inspección de Archivos:**
   - Verificación de cabeceras binarias (Magic Bytes) y tipo MIME real; no se confía únicamente en la extensión del archivo.
   - Límite máximo de tamaño de carga (ej. 25 MB por archivo) para evitar ataques de denegación de servicio por agotamiento de memoria.
2. **Mitigación de Macros y Scripts Office:**
   - El motor de plantillas OOXML (`docxtemplater`) procesa exclusivamente archivos XML internos de documentos `.docx`.
   - Se prohíbe la subida o ejecución de archivos con macros habilitadas (`.docm`).
   - La conversión a PDF mediante LibreOffice headless se ejecuta en workers con usuario de sistema sin privilegios administrativos, límites estrictos de memoria/CPU y sin acceso a la red externa.

---

## 6. Cumplimiento Normativo (Ley N.° 29733 - Perú)
- **Principio de Proporcionalidad y Finalidad:** La plataforma solo almacena datos personales de docentes (nombre, correo institucional, especialidad) indispensables para la autoría académica.
- **No reutilización para entrenamiento:** Ningún documento curricular, sílabo o contenido cargado por una institución se utilizará para entrenar o ajustar modelos de IA sin consentimiento explícito.
- **Derecho de Supresión / Cancelación:** Al darse de baja un instituto, se ofrece un proceso de purga demostrable de todos los registros y archivos almacenados tras el periodo de gracia configurado.
