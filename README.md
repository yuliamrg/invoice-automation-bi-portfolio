# Automatización de gestión de facturas (AppSheet + Apps Script + Looker Studio)

> Proyecto con enfoque **BI & Analytics**: automatización de captura y estructuración de datos desde facturas electrónicas (XML), trazabilidad documental en Drive, operación en AppSheet y visualización de indicadores en Looker Studio.

---

## Contenido
- [Contexto](#contexto)
- [Problema](#problema)
- [Objetivos](#objetivos)
- [Solución](#solución)
- [Arquitectura y flujo de datos](#arquitectura-y-flujo-de-datos)
- [Modelo de datos](#modelo-de-datos)
- [Implementación con Apps Script](#implementación-con-apps-script)
- [Capa operativa con AppSheet](#capa-operativa-con-appsheet)
- [BI con Looker Studio](#bi-con-looker-studio)
- [Impacto](#impacto)
- [Skills aplicados](#skills-aplicados)
- [Evidencias](#evidencias)
- [Estructura sugerida del repositorio](#estructura-sugerida-del-repositorio)
- [Próximos pasos](#próximos-pasos)

---

## Contexto

En mi rol como analista de mantenimiento debía gestionar las facturas asociadas al área. Las facturas llegaban por correo desde el área de recepción de facturas en un adjunto `.zip` que contenía:

- `.xml` (factura electrónica)
- `.pdf` (representación gráfica)

---

## Problema

El flujo original era manual y poco trazable:

1. Abrir el correo y descargar el `.zip`.
2. Descomprimirlo, abrir el **PDF** y validar si la factura correspondía a mantenimiento.
3. Responder al área de recepción (aceptar / rechazar / redirigir).
4. Si se aceptaba: imprimir, diligenciar formato de relación, radicar en contabilidad y guardar copia en folders.
5. Si alguien pedía trazabilidad, debía buscar en folders la evidencia.

**Dolores principales**
- Alto tiempo operativo por factura (descarga, revisión y respuesta).
- La información no quedaba consolidada → difícil control del gasto.
- Trazabilidad basada en búsquedas manuales (carpetas físicas/digitales).

---

## Objetivos

- Reducir el trabajo manual de descarga, revisión y registro.
- Estructurar datos desde el XML en una fuente única (Google Sheets).
- Automatizar, hasta donde fuera posible, la respuesta por correo (aceptación/rechazo/redirección).
- Mejorar la trazabilidad (correo ↔ soporte PDF/XML ↔ registro ↔ estado).
- Publicar indicadores en Looker Studio para control del gasto y presupuesto.

---

## Solución

Implementé un flujo integrado en **Google Workspace** con:

- **Gmail**: filtro/etiqueta para correos de facturas.
- **Google Apps Script**: proceso programado que:
  - busca hilos con una etiqueta,
  - descarga y descomprime ZIP,
  - guarda PDF/XML en Drive con estructura (proveedor/factura),
  - extrae campos del XML,
  - registra datos y links en Google Sheets,
  - registra errores en una hoja “Errores”.
- **Google Sheets**: dataset estructurado + trazabilidad (links).
- **AppSheet**: interfaz operativa para visualizar facturas y gestionar estados.
- **AppSheet Bot + Apps Script**: respuesta automática del correo según la acción elegida.
- **Looker Studio**: dashboard de indicadores (gasto, estados, proveedores, tiempo, presupuesto).

---

## Arquitectura y flujo de datos

### Flujo end-to-end

1. **Entrada**: correo con adjunto ZIP (XML + PDF).
2. **Clasificación**: filtro de Gmail → etiqueta (ej. `Facturas/Mantenimiento`).
3. **ETL ligero (Apps Script)**:
   - `GmailApp.search('label:...')` para obtener hilos.
   - Extrae ZIP y descomprime con `Utilities.unzip()`.
   - Identifica PDF/XML.
   - Crea carpetas en Drive por **Proveedor** y **No. Factura**.
   - Guarda archivos en Drive y obtiene URLs.
   - Lee el XML (factura electrónica) y extrae datos.
   - Inserta registro en Google Sheets (incluyendo links).
   - Evita duplicados validando `CUFE`.
   - Registra errores en una hoja “Errores”.
4. **Operación (AppSheet)**:
   - Lista de facturas con link directo al PDF.
   - Selección de estado (Aceptada/Rechazada/Redirigida).
   - Observaciones, responsable, confirmación, etc.
5. **Automatización de respuesta**:
   - Bot de AppSheet invoca script para responder el correo según el estado.
6. **BI (Looker Studio)**:
   - Consume el dataset y muestra KPIs y tendencias para control de gasto/presupuesto.

### Diagrama (placeholder)
- Agrega aquí tu diagrama de arquitectura:
  - `![Arquitectura](./images/arquitectura.png)`

---

## Modelo de datos

### Dataset principal: Google Sheets

Encabezado real del proyecto (fuente):

```
CUFE	FECHA REGISTRO	TIPO	FECHA FACTURA	NIT	PROVEEDOR	NO FACTURA	VALOR	LINK FACTURA	URL CORREO	ORDEN DE SERVICIO	URL ORDEN DE SERVICIO	PRESUPUESTO	ESTADO FACTURA	CONFIRMADA	CORREO ENVIADO	OBSERVACIONES	RESPONSABLE	DISTRIBUCION DE CENTRO DE COSTOS	ANIO	MES	DIA
```

**Notas de diseño**
- **CUFE**: clave única (usada para idempotencia; evita duplicados).
- **LINK FACTURA** y **URL CORREO**: trazabilidad inmediata (soporte + origen).
- **ANIO/MES/DIA**: soporte analítico temporal en Looker Studio.
- **ESTADO FACTURA**, **CONFIRMADA**, **CORREO ENVIADO**: control del proceso y auditoría.

> (Opcional) Puedes crear un diccionario de datos en `docs/diccionario_datos.md`.

---

## Implementación con Apps Script

### Enfoque técnico aplicado
- Lectura de Gmail por etiqueta para operar sobre una “bandeja controlada”.
- Procesamiento de adjuntos (ZIP → PDF/XML) y persistencia estandarizada en Drive.
- Extracción de datos desde XML para construir un dataset tabular.
- Validación de duplicados por CUFE (pipeline idempotente).
- Manejo de errores y registro (stack trace + link al correo).

### Fragmento representativo (función principal)

```js
//Función Principal
function procesarCorreosEtiquetados() {
  const threads = GmailApp.search('label:' + CONFIG.etiqueta);
  const parentFolder = DriveApp.getFolderById(CONFIG.carpetaPrincipalId);
  const sheet = getSheet(CONFIG.sheetName);
  const currentData = getCurrentData(sheet);
  const errorSheet = getSheet('Errores');

  let processed = 0;
  let errors = 0;

  for (let i = 0; i < threads.length; i++) {
    try {
      processThread(threads[i], sheet, currentData, parentFolder);
      processed++;
    } catch (error) {
      errors++;
      logError(errorSheet, error, i + 1, threads[i], '');
    }
  }
}
```

### Qué resuelve este módulo
- **Automatiza captura** de datos desde el correo (sin intervención manual).
- Centraliza información en Sheets y evidencia en Drive.
- Reduce el tiempo de “triage” al permitir consultar directamente los links.

> Placeholder para el repositorio (cuando lo publiques):
- Código Apps Script: `./src/appsscript/procesarCorreosEtiquetados.js`
- Helpers / utilidades: `./src/appsscript/helpers.js`

---

## Capa operativa con AppSheet

AppSheet se usa como la **interfaz** para operar el proceso:

- Visualización de facturas (lista/tabla).
- Apertura inmediata del PDF desde `LINK FACTURA`.
- Actualización de `ESTADO FACTURA` (Aceptada / Rechazada / Redirigida).
- Registro de `OBSERVACIONES`, `RESPONSABLE` y campos de control.
- Bot para automatizar la respuesta del correo según la acción seleccionada.

### Capturas / evidencias (placeholders)
- `![AppSheet - Lista](./images/appsheet_lista.png)`
- `![AppSheet - Detalle](./images/appsheet_detalle.png)`
- Link demo / descripción: **[Agregar enlace aquí](#)**

---

## BI con Looker Studio

Looker Studio consume el dataset (Sheets) para construir un tablero con indicadores de:

- Valor total de facturas procesadas.
- Cantidad de facturas.
- Valor promedio por factura.
- Distribución por estado (Aceptada / Rechazada / Redirigida).
- Top proveedores (treemap).
- Tendencia mensual (valor y número de facturas).
- Seguimiento de presupuesto usando el campo `PRESUPUESTO`.

### Capturas / evidencias (placeholders)
- `![Dashboard - Facturación](./images/dashboard_facturacion.png)`
- Link público (si aplica): **[Agregar enlace aquí](#)**

---

## Impacto

Cambios logrados:

- Eliminación de la descarga manual por factura para revisión inicial (validación desde links).
- Información consolidada en un dataset estructurado (Sheets) con trazabilidad.
- Facilita control del gasto mediante visualización (Looker Studio).
- Menor dependencia de copias impresas gracias a evidencia digital.

### Métricas (completa con datos reales)
- Tiempo promedio por factura **antes**: `__` min
- Tiempo promedio por factura **después**: `__` min
- Ahorro estimado semanal/mensual: `__` horas
- % facturas con trazabilidad completa (correo + pdf + registro): `__%`

---

## Skills aplicados

*(Solo habilidades evidentes en el alcance descrito y el código compartido)*

- **Google Apps Script (automatización)**
  - `GmailApp` (búsqueda/lectura de hilos por etiqueta)
  - `DriveApp` (carpetas/archivos, organización, URLs)
  - `SpreadsheetApp` (dataset en Sheets)
  - `Utilities.unzip` (procesamiento de ZIP)
  - Manejo de errores y logging (hoja de errores, stack trace)

- **ETL ligero / Data engineering en Google Workspace**
  - Extracción desde XML → transformación → carga en tabla (Sheets)
  - Idempotencia por `CUFE`
  - Estandarización de almacenamiento en Drive (Proveedor/Factura)

- **BI & Analytics**
  - Modelado tabular (dimensiones/medidas) para consumo analítico
  - Particiones temporales (`ANIO/MES/DIA`) para análisis por periodo
  - Dashboard en Looker Studio (KPIs, tendencias, distribución y top proveedores)

- **Low-code para operación**
  - AppSheet como capa operativa para gestión de estados y trazabilidad
  - Bot de AppSheet integrado con Apps Script para automatizar respuestas

---

## Evidencias

Agrega aquí lo que quieras mostrar en tu portafolio:

- 📷 Capturas: `./images/...`
- 🔗 Dashboard publicado: **[Link](#)**
- 🔗 App (AppSheet): **[Link](#)**
- 🧾 Ejemplo de registro en Sheets (sin datos sensibles): **[Link / Imagen](#)**

---

## Estructura sugerida del repositorio

```
/
├─ README.md
├─ images/
│  ├─ dashboard_facturacion.png
│  ├─ appsheet_lista.png
│  ├─ appsheet_detalle.png
│  └─ arquitectura.png
└─ src/
   └─ appsscript/
      ├─ procesarCorreosEtiquetados.js
      └─ helpers.js
```

---

## Próximos pasos

- Crear `docs/diccionario_datos.md` y documentar reglas de negocio.
- Agregar validaciones adicionales sobre campos obligatorios del XML.
- Documentar permisos mínimos requeridos en Google Workspace (Gmail/Drive/Sheets).
- (Si aumentara el volumen) evaluar una capa analítica más robusta manteniendo la operación en AppSheet.
