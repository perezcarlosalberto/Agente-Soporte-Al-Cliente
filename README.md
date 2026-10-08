# Agente de Soporte al Cliente con Python y Gemini

Agente de inteligencia artificial que lee automáticamente las quejas y sugerencias
registradas en un **Google Form**, las analiza con la **API de Gemini** (clasificación
y sentimiento) y guarda el resultado en **Google Sheets**, que funciona como base de datos.

Práctica *"Mi Primer Agente de IA"*: Desarrollo de Agentes Inteligentes, Unidad 2.

## Funcionamiento

```
Google Forms  ──►  Google Sheets  ──►  agente_soporte.py  ──►  Gemini  ──►  Google Sheets
 (entrada)        (respuestas A-C)       (el agente)          (análisis)    (resultados D-F)
```

Cada `INTERVALO_MINUTOS` minutos (1 minuto en este proyecto; la guía sugiere 15) el agente:

1. **Percibe:** lee todas las filas de la hoja de cálculo.
2. **Filtra:** omite las filas que ya tienen `SI` en la columna F (su "memoria").
3. **Razona:** envía cada comentario nuevo a Gemini, que devuelve un JSON con:
   - **Clasificación:** `Ventas`, `Soporte Técnico` o `Logística`
   - **Sentimiento:** `Positivo`, `Negativo` o `Neutro`
4. **Actúa:** escribe la clasificación (D), el sentimiento (E) y `SI` (F).

Si Gemini falla en una fila, el agente muestra el error y continúa con la siguiente.
La fila queda pendiente y se vuelve a intentar en el siguiente ciclo.

### Estructura de la hoja de cálculo

| Columna | Encabezado | La llena |
|---|---|---|
| A | Marca temporal | Google Forms |
| B | Tu Nombre | Google Forms |
| C | Describe tu queja, sugerencia o comentario | Google Forms |
| D | Clasificación IA | El agente |
| E | Sentimiento IA | El agente |
| F | Agente Ejecutado | El agente (`SI`) |

## Requisitos

- Python 3.9 o superior
- Una cuenta de Google (Forms, Sheets, Google Cloud Console y Google AI Studio)

```bash
pip install -r requirements.txt
```

| Librería | Uso |
|---|---|
| `gspread` | Leer y escribir en Google Sheets |
| `google-auth` | Autenticación con Google (OAuth 2.0) |
| `google-genai` | Conexión con la API de Gemini |
| `schedule` | Ejecución automática cada cierto tiempo |

## Configuración

### 1. Google Forms y Google Sheets
1. Crear el formulario **"Formulario de Quejas y Sugerencias"** con dos preguntas:
   *Tu Nombre* (respuesta corta) y *Describe tu queja, sugerencia o comentario* (párrafo).
2. En **Respuestas**, vincularlo a una hoja nueva llamada exactamente
   **`Respuestas de Quejas (Base de Datos)`**.
3. Agregar a mano los encabezados de las columnas D, E y F.

### 2. Google Cloud: credenciales OAuth
1. Crear un proyecto en [Google Cloud Console](https://console.cloud.google.com/).
2. Habilitar **Google Sheets API** y **Google Drive API**.
3. Configurar la pantalla de consentimiento OAuth como **Externa** y agregar tu
   correo en **Usuarios de prueba**.
4. Crear un **ID de cliente OAuth** de tipo **App de escritorio**, descargar el JSON
   y guardarlo como `client_secrets.json` en la carpeta del proyecto.

### 3. API Key de Gemini
1. Crear una clave en [Google AI Studio](https://aistudio.google.com/app/api-keys).
2. Copiar `config_local.example.py` como `config_local.py` y pegar la clave:

   ```python
   GEMINI_API_KEY = "tu_clave_aqui"
   ```

   También se puede usar la variable de entorno `GEMINI_API_KEY`.

## Ejecución

```bash
python agente_soporte.py
```

La primera vez se abre el navegador para iniciar sesión con Google y autorizar el
acceso a Sheets y Drive. El aviso *"Google no verificó esta app"* es normal: hay que
elegir **Continuar**. El token se guarda en `token_autorizado.json` y no se vuelve a pedir.

Salida de ejemplo:

```
============================================================
 Agente de Soporte iniciado
 Revisará 'Respuestas de Quejas (Base de Datos)' cada 1 minutos.
 Presiona Ctrl+C para detenerlo.
============================================================

[2026-10-07 12:17:02] Revisando nuevas quejas...
  Cliente: Ana
    Clasificación: Soporte Técnico
    Sentimiento:   Negativo
    -> Resultados escritos en Google Sheets (fila 3).
  Ciclo terminado: 1 queja(s) procesada(s).
```

## Pruebas

| Comentario | Resultado esperado |
|---|---|
| Mi pedido llegó tarde y la caja estaba rota. Muy mal servicio. | Logística / Negativo |
| No entiendo cómo funciona la nueva app, el botón de ayuda no sirve. | Soporte Técnico / Negativo |
| Me atendió un vendedor llamado Juan y fue increíble, ¡resolvió todo! 10/10. | Ventas / Positivo |
| Mi paquete lleva 2 semanas sin llegar y nadie me da información. | Logística / Negativo |

## Estructura del proyecto

```
├── agente_soporte.py         # Script del agente
├── config_local.example.py   # Plantilla para la API Key de Gemini
├── requirements.txt          # Dependencias de Python
├── .gitignore                # Excluye credenciales y archivos locales
└── README.md
```

Archivos locales que **no** se suben al repositorio:

| Archivo | Contenido |
|---|---|
| `client_secrets.json` | Credenciales OAuth de Google Cloud |
| `token_autorizado.json` | Token de acceso generado al autorizar |
| `config_local.py` | API Key de Gemini |

## Notas

- **Modelo:** la guía original usa `gemini-2.5-flash`, pero Google ya no lo ofrece a
  usuarios nuevos. El agente usa `gemini-3.1-flash-lite`, configurable en `MODELO_GEMINI`.
- **Reintentos:** si Gemini está saturado (errores 429, 503 o 504), la petición se
  reintenta hasta 5 veces antes de dar la fila por fallida.
- **Limitaciones:** Gemini siempre elige una de las tres categorías, así que los
  comentarios que no encajan en ninguna (o que mezclan varios temas) se asignan a la
  más cercana.

## Solución de errores comunes

| Error | Solución |
|---|---|
| `FileNotFoundError: client_secrets.json` | Colocar el archivo junto a `agente_soporte.py` |
| `Error 403: access_denied` | Agregar el correo en **Usuarios de prueba** de OAuth |
| `SpreadsheetNotFound` | Revisar que el nombre de la hoja coincida exactamente con `NOMBRE_HOJA` |
| `RefreshError: invalid_grant` | Borrar `token_autorizado.json` y volver a ejecutar |
| `API key not valid` | Copiar de nuevo la clave desde AI Studio, sin espacios |
| `404 NOT_FOUND` (modelo) | Cambiar `MODELO_GEMINI` por un modelo disponible en AI Studio |
| `503 UNAVAILABLE` | Gemini está saturado; el agente reintenta en el siguiente ciclo |
