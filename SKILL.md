---
name: recordatorio-telegram
description: Ayuda a configurar y personalizar un recordatorio automático enviado a Telegram mediante una tarea programada en GitHub Actions. Úsalo cuando alguien diga "quiero un recordatorio en Telegram", "que me avise a tal hora por Telegram", "configura mi bot de recordatorios", "cambia la hora de mi recordatorio", "agrega un recordatorio", o cuando quiera adaptar el paquete recordatorios-telegram (script enviar_recordatorio.py + recordatorios.json + workflow de GitHub Actions). Cubre los tres modos: mensaje fijo, lista de recordatorios y recordatorio generado con IA (API de Claude).
---

# Configurar un recordatorio por Telegram (GitHub Actions)

Esta skill guía a un colaborador para poner en marcha o personalizar el paquete **recordatorios-telegram**: una tarea programada que envía recordatorios a Telegram sin servidor propio. Está basada en el patrón del skill `integraciones-agentes` del proyecto *Agentes de negocio* (Dra. Elda C. Morales).

## Cuándo usar esta skill
- El usuario quiere recibir avisos automáticos en Telegram a una hora que pueda modificar.
- Ya tiene el paquete `recordatorios-telegram` (o quiere crearlo) y necesita: crear el bot, obtener el chat_id, subirlo a GitHub, configurar los Secrets, ajustar la hora o editar los recordatorios.
- Quiere entender o activar los recordatorios de tipo IA.

## Regla de oro (seguridad)
Nunca escribas el token de Telegram ni la API key dentro del código, del JSON, del workflow ni del chat. Las credenciales van **solo** en el `.env` local (que está en `.gitignore`) o en **GitHub → Settings → Secrets and variables → Actions**. Si una llave llegó a git, se considera comprometida: hay que rotarla con @BotFather / en la consola, no solo borrarla.

## Arquitectura del paquete
```
recordatorios-telegram/
├── enviar_recordatorio.py     # arma y envía los mensajes (Telegram Bot API)
├── obtener_chat_id.py         # utilidad para descubrir el chat_id
├── recordatorios.json         # QUÉ se envía, QUÉ días  (capa 2 del horario)
├── requirements.txt
├── .env.example               # plantilla de credenciales (uso local)
├── .gitignore                 # excluye .env
└── .github/workflows/recordatorio.yml   # CUÁNDO (la HORA, cron en UTC)
```
Dos capas de horario: el `cron` del workflow decide la **hora**; el campo `dias` de cada recordatorio decide **qué días** y el `mensaje`/`prompt` decide **qué dice**.

## Flujo de trabajo a seguir con el usuario

Antes de tocar archivos, confirma con el usuario: (a) ¿a qué **hora local** quiere el aviso?, (b) ¿qué **días**?, (c) ¿el mensaje será **fijo**, una **lista** de varios, o **con IA**?, (d) ¿ya tiene bot de Telegram y cuenta de GitHub?

Luego guíalo en este orden (dale instrucciones concretas, no supongas conocimientos técnicos):

1. **Crear el bot**: hablar con **@BotFather** en Telegram → `/newbot` → obtener el **token**.
2. **Obtener el chat_id**: pedirle que le escriba `hola` a su bot; luego abrir `https://api.telegram.org/bot<TOKEN>/getUpdates` y leer `chat.id`. (O ejecutar `python obtener_chat_id.py`.)
3. **Subir el paquete** a un repositorio de GitHub (puede ser privado). Recordar incluir la carpeta `.github`. Nunca subir `.env`.
4. **Crear los Secrets** en GitHub con nombres exactos: `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID` y, solo si usará IA, `ANTHROPIC_API_KEY`.
5. **Ajustar la hora** en `.github/workflows/recordatorio.yml`, línea `cron`. El cron está en **UTC**; para CDMX (UTC-6) se **suman 6 horas** a la hora local. Ej.: 08:00 CDMX → `"0 14 * * *"`.
6. **Editar los recordatorios** en `recordatorios.json` (ver plantillas abajo).
7. **Probar** con **Actions → Run workflow** (ejecución manual inmediata) y confirmar que llega el mensaje.

## Plantillas de recordatorios.json

Mensaje fijo:
```json
{ "id": "revisar-pipeline", "tipo": "fijo", "activo": true,
  "dias": ["lun","mie","vie"],
  "mensaje": "🔔 Revisa el pipeline y los pendientes del día." }
```

Lista: simplemente varios bloques como el anterior, separados por comas, dentro de `"recordatorios": [ ... ]`.

Con IA (lo redacta Claude en cada ejecución):
```json
{ "id": "arranque-ia", "tipo": "ia", "activo": true, "dias": ["lun"],
  "prompt": "Escribe un recordatorio breve (máx 3 líneas) para arrancar la semana con foco. Incluye una micro-acción concreta.",
  "modelo": "claude-haiku-4-5", "max_tokens": 300,
  "mensaje": "🚀 Define tu prioridad #1 y empieza por ahí." }
```
En el tipo IA, `mensaje` es el **respaldo** que se envía si la API falla. Requiere `ANTHROPIC_API_KEY`. Advierte al usuario que la IA tiene un **costo pequeño** y que debe verificar el **nombre de modelo y el precio vigentes** en la documentación oficial de Claude antes de operar.

## Cómo convertir la hora local a cron UTC
`cron: "minuto hora * * *"` en UTC. Para CDMX: hora_UTC = hora_local + 6.
- 07:00 → `"0 13 * * *"` · 08:00 → `"0 14 * * *"` · 12:00 → `"0 18 * * *"` · 20:30 → `"30 2 * * *"`.
Se pueden poner varias líneas de `cron` para varios horarios. Los cron de Actions pueden retrasarse algunos minutos.

## Errores frecuentes (para diagnosticar rápido)
- No llega nada → el usuario no le escribió primero al bot, o el `chat_id` está mal.
- `ERROR: faltan TELEGRAM_BOT_TOKEN...` → Secret ausente o mal nombrado.
- El de IA no llega → falta `ANTHROPIC_API_KEY` o cambió el nombre del modelo.
- Llega a otra hora → confusión UTC vs. local (recuerda +6 para CDMX).

## Validación final
- [ ] `Run workflow` manual entregó el mensaje en Telegram.
- [ ] El `cron` corresponde a la hora local deseada (+6 a UTC).
- [ ] Los recordatorios activos tienen los días correctos.
- [ ] `.env` no está en el repositorio; las llaves están solo en Secrets.

## Cierre recomendado al usuario
- **Recomendación principal:** GitHub Actions — gratis, sin máquina encendida, horario editable, fácil de compartir (usar el repo como *Template*).
- **Alternativa sencilla:** `python enviar_recordatorio.py --simular` para ver los mensajes sin credenciales.
- **Alternativa profesional:** desplegar junto a un bot interactivo en Render/VPS con control de costos si se usa IA.

## Fuentes
- Telegram Bot API: https://core.telegram.org/bots/api
- Modelos de Claude: https://platform.claude.com/docs/en/about-claude/models/overview
- Precios de la API de Claude: https://docs.anthropic.com/en/docs/about-claude/pricing
- GitHub Actions (cron/schedule): https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#schedule
