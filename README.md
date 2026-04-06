# Proyecto Final — Automatización con n8n y LLMs
Análisis Automatizado de Sentimiento de Tweets


Paso 1 — Importar el workflow en n8n

1. Abrir n8n en el navegador.
2. En la aprte superior derecha, hacer clic en **Create workflow**.
3. En la nueva pantalla, hacer clic en los tres puntos que también se encuentran en la parte superior derecha y luego en **Import from file**.
4. Seleccionar el archivo **workflow.json**.
5. El workflow se importa con todos los nodos y conexiones. Aparecerá en estado **inactivo**.


Paso 2 — Configurar credenciales

El workflow requiere tres credenciales. Ninguna clave real está incluida en el JSON entregado.

OpenAI
1. Ir al tercer nodo, **OpenAI Clasificador**, y seleccionar tu credencial **OpenAI** en el campo **Credential to connect with** (si ya tenés) o **Create new credential** (si aún no la hiciste).
2. En caso de crear una nueva, se hará desde **https://api.openai.com/v1**, obteniendo la API Key necesaria para el funcionamiento.
3. Seleccionar también esta credencial en los nodos **OpenAI Explicador** y **OpenAI Draft**.

Google Sheets
1. Ir al primer nodo, **Trigger**, y seleccionar tu credencial en el campo **Credential to connect with**.
2. Autenticarse con OAuth2 (se abre una ventana de Google para autorizar). En caso de no tener esta credencial, deberás crear un permiso para utilizarla en **https://console.cloud.google.com/**
3. Especificar la **hoja de cálculo** que vas a utilizar **(Document)** y la hoja de la que se debe tomar la información **(Sheet)**.

Gmail
1. Ir al nodo **Enviar Email**, y seleccionar tu credencial **Gmail OAuth2 API** en el campo **Credential to connect with** (si ya tenés) o **Create new credential** (si aún no la hiciste). Para crearla, el proceso es muy similar que el nodo de OpenAI.
2. Modificá el campo **To** con el mail al que querés recibir los correos de alerta, cambiando **YOUR_EMAIL@gmail.com** por el correcto.


Paso 3 — Configurar Google Sheets

1. Creá un archivo de **Hojas de cálculo** desde las herramientas de Google Sheets. Dentro de este archivo, la hoja en la que vas a escribir se va a llamar Hoja1. En esta hoja debés escribir en la primera casilla solamente la palabra **tweet** (en minúscula).
2. Agregar otra hoja dentro del mismo archivo, esta deberá llamarse **Métricas**. Debe contener las siguientes columnas: **timestamp**, **tweet**, **sentiment** y **topic**.
3. Copiar el ID del archivo de Google Sheets desde la URL: `https://docs.google.com/spreadsheets/d/ESTE_ES_EL_ID/edit`
4. En el nodo **Trigger** y en el nodo **Guardar Metricas**, reemplazar `YOUR_GOOGLE_SHEET_ID` por ese ID.


Paso 4 — Activar el workflow

1. Hacer clic en el botón **Publish** (esquina superior derecha).
2. El workflow quedará escuchando nuevas filas en Google Sheets cada minuto.


Paso 5 — Prueba

Agregar cada tweet como una nueva fila en la columna **tweet** de la Hoja1 y esperar hasta 1 minuto para ver el resultado.

Tweet de prueba 1 — NEGATIVO (esperado)
```
Pongan a todos 6 horas seguidas a patear al arco. Me tienen podrido que no se les caiga ni un gol. 700 chances por partido y no entran? Para ganar hay que hacer los goles
```
**Resultado esperado:**
- Clasificación: `NEGATIVO`
- Topic: `fútbol` o `deportes`
- Se activa LLM Explicador → LLM Draft → se envía email de alerta
- Se guarda fila en hoja Métricas

Tweet de prueba 2 — NEUTRAL (esperado)
```
Compré una cosa por Shein y estoy esperando que me llegue, veremos como sale
```
**Resultado esperado:**
- Clasificación: `NEUTRAL`
- Topic: `entrega` o `pedido`
- No se envía email de alerta
- Se guarda fila en hoja Métricas

Tweet de prueba 3 — POSITIVO (esperado)
```
¡Me encanta mi trabajo!
```
**Resultado esperado:**
- Clasificación: `POSITIVO`
- Topic: `trabajo`
- No se envía email de alerta
- Se guarda fila en hoja Métricas



## Estructura del flujo

[Trigger: Google Sheets]
        ↓
[Inicio: validar campo tweet]
        ↓
[Chain Clasificador + OpenAI] → JSON {sentiment, topic}
        ↓
[Extraer sentimiento]
        ↓
[Necesita reintento?]
   ↓ SÍ                    ↓ NO
[Reintento]           [Es negativo?]
   ↓ (vuelve al       ↓ SÍ              ↓ NO
    Clasificador)  [Chain Explicador] [Guardar Métricas]
                       ↓                    ↓
                [Preparar Datos]      [Log Positivo]
                       ↓
            [Chain Draft Community]
                       ↓
                [Extraer Draft]
                       ↓
                [Enviar Email]
                       ↓
               [Guardar Métricas]



Manejo de errores

Si LLM devuelve formato inesperado → El nodo Code intenta parsing JSON; si falla, aplica fallback por palabras clave
Si el sentimiento no es reconocible → Nodo **Necesita reintento** reintenta hasta 3 veces
Si el campo tweet está vacío → El nodo **Inicio** lanza excepción con mensaje descriptivo
Si falla en Enviar Email → El n8n registra el error en el log de ejecuciones


Notas de seguridad

- No se incluyen credenciales reales en ningún archivo entregado.
- Las API Keys de OpenAI y los tokens OAuth de Google se almacenan exclusivamente en la base de datos interna de n8n.
- El JSON del workflow solo contiene IDs internos de referencia, que no funcionan fuera de la instancia n8n original.
