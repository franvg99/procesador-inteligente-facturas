# 🧾 Procesador Inteligente de Facturas

![Estado del Proyecto](https://img.shields.io/badge/Status-Finalizado-success)
![n8n](https://img.shields.io/badge/Workflow-n8n-orange)
![AI](https://img.shields.io/badge/AI-OpenAI-black)

Este proyecto implementa un **flujo automatizado en n8n** que monitorea una casilla de **Gmail**, detecta emails con facturas de proveedores en PDF, extrae los datos fiscales relevantes utilizando **IA** (GPT-4o mini para PDFs digitales y GPT-4o Vision para documentos escaneados) y los registra automáticamente en **Google Sheets** sin intervención humana.

---

## Estructura del Repositorio

```text
├── README.md                        # Documentación técnica y funcional
├── workflow/
│   └── workflow_n8n.json            # Archivo fuente para importar en n8n
├── prompts/
│   ├── prompt_pdf_digital.txt       # Prompt para extracción de PDFs con texto
│   └── prompt_pdf_escaneado.txt     # Prompt para extracción de PDFs escaneados (Vision)
└── evidencia/
    ├── workflow_completo.png        # Vista general del canvas en n8n
    ├── seccion_recepcion.png        # Nodos de recepción y separación de documentos
    ├── seccion_filtros.png          # Nodos de filtrado y validación
    ├── seccion_normalizacion.png    # Nodo de normalización de binarios
    ├── seccion_ia.png               # Nodos de procesamiento con IA
    ├── seccion_validacion.png       # Nodos de validación de JSON
    ├── seccion_destino.png          # Nodos de escritura en Google Sheets
    ├── seccion_errores.png          # Error Workflow (flujo separado)
    ├── sheets_log.png               # Hoja de facturas con datos reales
    └── dashboard_looker.png         # Dashboard de Looker Studio conectado a Sheets
```

---

## Funcionalidades principales

- 📥 Monitoreo automático de **Gmail** cada minuto en busca de emails con PDF adjuntos
- 📄 Soporte para **PDFs digitales** (con texto extraíble) y **PDFs escaneados** (OCR visual)
- 🤖 Uso de **IA** para extracción de campos fiscales argentinos:
  - Tipo de comprobante (A, B o C)
  - Número de comprobante
  - Fechas de emisión y vencimiento
  - Razón social y CUIT del proveedor y receptor
  - Condición de IVA de ambas partes
  - Importes: neto gravado, IVA discriminado, otros impuestos, IVA contenido, total
  - CAE y vencimiento CAE
  - Moneda
- 📊 Registro automático en **Google Sheets** con estructura de 19 columnas
- 🚨 Sistema de **manejo de errores en tres niveles**: try/catch, hoja de errores y Error Workflow con notificación por Gmail y registro en Notion
- 📈 Compatible con **Google Looker Studio** para visualización de datos

---

## Lógica del flujo

1. **Gmail Trigger**
   Monitorea la bandeja de entrada cada 1 minuto. Solo procesa emails con adjuntos PDF.

2. **Dividir Documentos**
   Separa cada PDF adjunto en un ítem individual. Si el email tiene múltiples PDFs, el flujo los procesa en paralelo.

3. **Extraer datos binarios**
   Convierte el PDF binario a texto plano para su posterior análisis.

4. **Filtros de Validación**
   - *¿PDF o SCAN?* verifica si el PDF tiene texto extraíble (más de 100 caracteres).
   - *¿Es factura?* verifica que el documento contenga la palabra "FACTURA" — descarta recibos, presupuestos y otros documentos.

5. **Normalización de Binarios** *(rama escaneados)*
   Estandariza el nombre del binario a `attachment_0` para compatibilidad con el nodo GPT-4o Vision.

6. **Procesamiento con Inteligencia Artificial**
   - *GPT-4o mini* analiza el texto extraído del PDF digital y devuelve un JSON estructurado.
   - *GPT-4o Vision* recibe el PDF escaneado como imagen, lo lee visualmente y aplica el mismo esquema de extracción.

7. **Validación**
   - *JSON Válido* convierte el texto del LLM en un objeto JSON. Si el parseo falla, captura el error sin interrumpir el resto del flujo.
   - *¿Es válida?* verifica que el campo `error` esté vacío y que `tipo_comprobante` no sea nulo antes de escribir en Sheets.

8. **Destino Final**
   - *Insertar Factura* escribe una fila nueva en la hoja principal de Google Sheets.
   - *Insertar Error* registra los documentos que no pudieron procesarse en una hoja separada para auditoría.

9. **Error Workflow** *(flujo separado)*
   Se activa automáticamente ante cualquier falla crítica. Envía una notificación por email al administrador y registra el error en Notion con estado `Pendiente`.

---

## Tecnologías Utilizadas

* **Orquestador:** [n8n](https://n8n.io/) (self-hosted en Docker)
* **Inteligencia Artificial:** OpenAI API — `gpt-4o-mini` y `gpt-4o` (Vision)
* **Trigger:** Gmail API
* **Base de Datos:** Google Sheets API
* **Monitoreo:** Notion API
* **Seguridad:** Google Cloud Platform (OAuth 2.0 Client ID & Secret)
* **Visualización:** Google Looker Studio

---

## Campos extraídos

El flujo extrae y registra los siguientes 19 campos por factura:

| Campo | Descripción |
|---|---|
| `tipo_comprobante` | Letra del comprobante AFIP (A, B o C) |
| `numero_comprobante` | Número en formato 0000-00000000 |
| `fecha_emision` | Fecha de emisión del comprobante |
| `fecha_vencimiento` | Fecha de vencimiento del comprobante |
| `proveedor_razon_social` | Razón social del emisor |
| `proveedor_cuit` | CUIT del emisor |
| `proveedor_condicion_iva` | Condición frente al IVA del emisor |
| `receptor_nombre` | Nombre o razón social del receptor |
| `receptor_cuit_dni` | CUIT o DNI del receptor |
| `receptor_condicion_iva` | Condición frente al IVA del receptor |
| `condicion_venta` | Condición de venta (contado, cuenta corriente, etc.) |
| `importe_neto_gravado` | Base imponible |
| `iva_discriminado` | IVA discriminado en el comprobante |
| `otros_impuestos` | Percepciones u otros tributos |
| `iva_contenido` | IVA incluido en el precio (comprobantes B/C) |
| `importe_total` | Total del comprobante |
| `cae` | Código de Autorización Electrónica |
| `vencimiento_cae` | Fecha de vencimiento del CAE |
| `moneda` | Moneda del comprobante (ARS, USD, etc.) |

---

## Evidencias de Funcionamiento

### 1. Vista General del Workflow
Arquitectura completa del flujo en n8n con sus secciones documentadas mediante Sticky Notes.

![Workflow Completo](evidencia/workflow_completo.png)

### 2. Recepción y Separación de Documentos
Gmail Trigger, Split Out y Extract from File.

![Recepción](evidencia/seccion_recepcion.png)

### 3. Filtros de Validación
Detección de PDF digital vs escaneado e identificación de comprobantes fiscales.

![Filtros](evidencia/seccion_filtros.png)

### 4. Normalización de Binarios
Estandarización del nombre del adjunto para compatibilidad con GPT-4o Vision.

![Normalización](evidencia/seccion_normalizacion.png)

### 5. Procesamiento con Inteligencia Artificial
GPT-4o mini para PDFs digitales y GPT-4o Vision para documentos escaneados.

![IA](evidencia/seccion_ia.png)

### 6. Validación
Parseo del JSON y verificación de comprobante válido antes de escribir en Sheets.

![Validación](evidencia/seccion_validacion.png)

### 7. Destino Final
Escritura en Google Sheets — hoja de facturas válidas y hoja de errores para auditoría.

![Destino](evidencia/seccion_destino.png)

### 8. Manejo de Errores
Error Workflow con notificación por Gmail y registro en Notion.

![Errores](evidencia/seccion_errores.png)

### 9. Log de Facturas en Google Sheets
Registro estructurado de facturas procesadas con los 19 campos fiscales.

![Google Sheets](evidencia/sheets_log.png)

### 10. Dashboard en Looker Studio
Visualización del total facturado, IVA, ranking de proveedores y evolución temporal, conectado directamente a la planilla de Sheets.

![Dashboard](evidencia/dashboard_looker.png)

---

## Cómo ejecutar este proyecto

1. Tener una instancia de **n8n** corriendo (self-hosted con Docker o n8n Cloud).
2. Importar el archivo `workflow/workflow_n8n.json` desde el menú de n8n.
3. Configurar las credenciales en n8n:
   * **Gmail:** OAuth 2.0 via Google Cloud Console (Gmail API habilitada)
   * **Google Sheets:** OAuth 2.0 via Google Cloud Console (Sheets API y Drive API habilitadas)
   * **OpenAI:** API Key desde platform.openai.com
   * **Notion:** Integration Token desde notion.so/my-integrations
4. Crear la planilla **Facturas Proveedores** en Google Sheets con dos hojas: `Facturas` y `Errores`, con los encabezados correspondientes a los 19 campos.
5. Crear la base de datos **Sistema de Monitoreo** en Notion con las propiedades: Nombre workflow (Title), Fecha (Date), Error (Text), Detalle (Text), Estado (Select).
6. Vincular el Error Workflow al flujo principal desde Settings → Error Workflow.
7. Activar el workflow principal.

---

## Seguridad y Configuración

* No se utilizan Service Accounts — se implementó **OAuth 2.0** a través de Google Cloud Console.
* Las credenciales nunca se almacenan en el repositorio — usar el archivo `.env.example` como referencia.
* El flujo opera con **mínimo privilegio**: acceso de escritura solo sobre la planilla creada para el proyecto.

---

## Variables y Credenciales

Copiá el archivo `.env.example` como `.env` y completá los valores antes de importar el workflow:

```env
OPENAI_API_KEY=
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
NOTION_INTEGRATION_TOKEN=
```

---

## Autor

**Desarrollado por Franco Valentin Guerrero**

* 🤖 **Especialidad:** Consultoría de IA & Automatización para PYMEs
* 🐙 **GitHub:** [Ver perfil](https://github.com/franvg99)
* 💼 **LinkedIn:** [Ver perfil](https://linkedin.com/in/fguerrero99)
