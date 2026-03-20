# bot-gastos

Agente conversacional de gastos personales vía Telegram. Reemplaza el bot-gastos V2 con comprensión de lenguaje natural, memoria de categorías aprendidas, soporte para fechas relativas, consultas en lenguaje libre, edición y borrado de entradas anteriores. Los datos se guardan en el mismo Google Sheet que el V2.

---

## Requisitos

- Python 3.11+
- Dependencias: ver `requirements.txt`
- Credenciales Google Service Account (reutilizar las del V2)
- Bot de Telegram configurado
- API key de Anthropic (Claude Sonnet)

---

## Instalación y setup

```bash
# En el servidor
cd 
chmod +x setup.sh
./setup.sh

# Completar .env
cp .env.example .env
chmod 600 .env
nano .env

# Copiar credenciales Google
cp bot-gastos/bot-gastos-*.json \
   /ruta/al/bot-gastos-credentials.json
```

### Variables de entorno (`.env.example`)

```
TELEGRAM_BOT_TOKEN="..."
TELEGRAM_CHAT_ID="..."
ANTHROPIC_API_KEY="..."
GOOGLE_SHEET_ID="..."
GOOGLE_CREDENTIALS_FILE="/ruta/al/bot-gastos-credentials.json"
```

---

## Uso manual

```bash
cd bot-gastos-v3
./venv/bin/python bot.py
```

> Probar manualmente antes de activar systemd.

---

## Uso con systemd

```bash
sudo cp bot-gastos-v3.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable bot-gastos-v3
sudo systemctl start bot-gastos-v3
```

---

## Comandos de operación y diagnóstico

```bash
# Estado
sudo systemctl status bot-gastos-v3

# Logs en tiempo real
sudo journalctl -u bot-gastos-v3 -f

# Ver tokens consumidos por llamada
sudo journalctl -u bot-gastos-v3 -f | grep Tokens

# Últimos 50 logs
sudo journalctl -u bot-gastos-v3 -n 50 --no-pager

# Reiniciar
sudo systemctl restart bot-gastos-v3

# Después de cambiar .env
sudo systemctl daemon-reload && sudo systemctl restart bot-gastos-v3
```

---

## Archivos clave

| Archivo | Qué contiene |
|---|---|
| `bot.py` | Agente principal, handlers Telegram, confirmaciones |
| `sheets.py` | Lectura, escritura, edición y borrado en Google Sheets |
| `memoria.py` | Gestión de memoria.json |
| `system_prompt.py` | System prompt del agente Claude |
| `memoria.json` | Asociaciones concepto→categoría aprendidas (nunca en git) |
| `.env` | Credenciales (nunca en git) |
| `bot-gastos-v3.service` | Definición systemd |

---

## Uso del bot

### Registrar un gasto
```
800 café
gasté 50000 en ropa ayer
60000 en 6 cuotas zapatillas
pagué 15000 de farmacia el 10/3
5 lucas en cigarrillos
```

### Consultas en lenguaje natural
```
¿cuánto gasté esta semana?
¿cuánto gasté en supermercado este mes?
resumen de marzo
¿cuánto gasté ayer en total?
```

### Editar entradas anteriores
```
el sushi del 10/3 era $25.000
cambiá el café de ayer a Hogar
el alquiler de marzo fue $180.000
modificar fecha apple 16/3 por 17/3
```

### Borrar gastos
```
borrar el café de hoy
borrar paseador del 16/3
mostrar gastos del 14/3 para elegir cual borrar
```

### Comandos
```
/ayuda       — instrucciones
/reset       — limpia el historial de conversación (no afecta Sheets ni memoria)
/memoria     — muestra las asociaciones concepto→categoría aprendidas
```

---

## Arquitectura del agente

```
Mensaje usuario
      ↓
Claude (Sonnet) con:
  - system prompt
  - historial últimos 15 mensajes
  - fecha/hora ART (calculada en cada llamada)
  - memoria.json
      ↓
JSON estructurado { tipo, datos, mensaje_usuario }
      ↓
┌──────────────────────────────────────────────┐
│ tipo=registro  → confirmación inline + cats  │
│ tipo=consulta  → leer Sheets → Claude        │
│ tipo=edicion   → buscar fila → confirmar     │
│ tipo=borrado   → buscar/listar → confirmar   │
│ tipo=categoria → agregar/eliminar            │
└──────────────────────────────────────────────┘
      ↓
Google Sheets (mismo sheet que V2)
```

---

## Memoria de categorías

`memoria.json` guarda las correcciones del usuario:
```json
{
  "regalos": "Personales",
  "sushi": "Hormiga",
  "vitaminas": "Personales"
}
```

Se actualiza automáticamente cuando el usuario cambia una categoría en la confirmación. Se incluye en cada llamada a Claude.

---

## Google Sheets — estructura

Una hoja por mes, nombre `YYYY-MM`.

| Fecha | Categoría | Concepto | Monto | Cuota | Tipo |
|---|---|---|---|---|---|
| 2026-03-14 | Hormiga | café | 800 | | real |
| 2026-03-14 | Deudas/Cuotas | zapatillas | 60000 | 1/6 | real |
| 2026-04-14 | Deudas/Cuotas | zapatillas | 60000 | 2/6 | informativo |

- `real`: suma al total del mes
- `informativo`: cuotas futuras, no suman (evitan doble contabilidad)

---

## Anti-duplicado

Si Google Sheets da timeout pero igualmente procesó el `append_row`, el bot detecta la fila existente (misma fecha + concepto + monto) y omite el reintento. Evita duplicados por problemas de red.

---

## Checklist — está vivo

- [ ] `systemctl status bot-gastos-v3` → active (running)
- [ ] `journalctl -u bot-gastos-v3 -n 10` → sin errores recientes
- [ ] Bot responde a `/start` en Telegram
- [ ] Un gasto de prueba aparece en la Sheet con fecha correcta
- [ ] `memoria.json` existe y es JSON válido

---

## Troubleshooting

| # | Síntoma | Causa probable | Solución |
|---|---|---|---|
| 1 | `ModuleNotFoundError` | venv no activado o deps no instaladas | `./venv/bin/pip install -r requirements.txt` |
| 2 | `Invalid API key` | ANTHROPIC_API_KEY mal configurada | Verificar `.env` |
| 3 | `Unauthorized` en Telegram | Token incorrecto | Verificar TELEGRAM_BOT_TOKEN en `.env` |
| 4 | Respuesta no JSON de Claude | Model devolvió texto libre | Ver logs — ocurre raramente con prompts muy ambiguos |
| 5 | `WorksheetNotFound` | La hoja del mes no existe aún | El bot la crea automáticamente al primer registro |
| 6 | Bot ignora mensajes | TELEGRAM_CHAT_ID incorrecto | Verificar el chat_id en `.env` |
| 7 | `systemd status = failed` | Error al iniciar | `journalctl -u bot-gastos-v3 -n 20` |
| 8 | `.env` no cargado en systemd | Falta `EnvironmentFile=` en el `.service` | Revisar el archivo `.service` |
| 9 | Gasto duplicado en Sheets | Timeout de red en Google | Bot detecta y omite automáticamente en el reintento |
| 10 | Error al editar fecha | Versión de gspread incompatible | Verificar que sheets.py usa `update()` no `update_cell()` |

---

## Seguridad

- `.env` fuera de git (`chmod 600 .env`)
- `bot-gastos-credentials.json` fuera de git
- `memoria.json` fuera de git (datos personales)
- Bot solo responde al CHAT_ID configurado
- Historial de conversación en RAM — no se persiste en disco
- API key de Anthropic dedicada a este proyecto

---

## Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-03-14 | v3.0.0 — versión inicial del agente conversacional |
| 2026-03-14 | v3.0.1 — parser JSON robusto con 3 intentos en cascada |
| 2026-03-14 | v3.0.1 — tono rioplatense + lunfardo monetario en system prompt (lucas, palos, K, M) |
| 2026-03-14 | v3.0.1 — parse_monto() para formato argentino (punto=miles, coma=decimal) |
| 2026-03-14 | v3.0.1 — log de tokens input/output en journalctl por llamada a Claude |
| 2026-03-14 | v3.0.1 — validado en producción: saludos, consultas, ediciones OK |
| 2026-03-14 | v3.0.2 — fix fechas con apóstrofe en registros y ediciones (evita número serial en Sheets) |
| 2026-03-14 | v3.0.2 — fix ediciones múltiples: el agente las procesa de a una con aviso al usuario |
| 2026-03-16 | v3.0.3 — borrado de gastos: borrar fila de Sheets en lenguaje natural con confirmación |
| 2026-03-16 | v3.0.3 — fix apóstrofe en fechas: value_input_option RAW en append_row |
| 2026-03-16 | v3.0.4 — fix borrado: elif tipo borrado faltaba en manejar_mensaje |
| 2026-03-16 | v3.0.4 — fix borrado sin concepto: listar gastos del día con botones inline |
| 2026-03-16 | v3.0.4 — fix formato monto en confirmación de borrado |
| 2026-03-16 | v3.0.5 — fix buscar_fila_para_editar filtra por fecha exacta |
| 2026-03-16 | v3.0.5 — fix botones borrado muestran fecha + concepto + monto |
| 2026-03-16 | v3.0.5 — fix anti-duplicado: fila_ya_existe() verifica antes de append_row |
| 2026-03-16 | v3.0.5 — fix doble tap en ✅ (error message not modified ignorado) |
| 2026-03-18 | v3.0.6 — fix editar_fila: update() en vez de update_cell() para compatibilidad gspread |
| 2026-03-18 | v3.0.6 — fix fecha en nuevos registros (tomaba fecha del historial en RAM) |
