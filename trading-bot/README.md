# trading-bot

Bot de trading automático para Binance Spot. Corre como servicio systemd en Linux.
Operá el par y la estrategia que quieras — todo se configura desde el `.env` sin tocar el código.

---

## Requisitos

- Python 3.10+
- Cuenta en Binance (Spot) con API habilitada
- Bot de Telegram configurado ([BotFather](https://t.me/BotFather)) — para notificaciones
- Linux con systemd (Ubuntu, Lubuntu, Debian, etc.)

---

## Instalación

```bash
git clone https://github.com/tu-usuario/trading-bot
cd trading-bot

python3 -m venv venv
./venv/bin/python -m pip install -U pip wheel
./venv/bin/pip install -r requirements.txt

cp .env.example .env
nano .env        # completar con tus credenciales y parámetros
chmod 600 .env
```

### Variables de entorno (`.env.example`)

```
BINANCE_API_KEY="..."
BINANCE_API_SECRET="..."
TELEGRAM_BOT_TOKEN="..."
TELEGRAM_CHAT_ID="..."

SYMBOL="ETHUSDC"           # par a operar
TIMEFRAME="1h"             # vela de referencia
CAPITAL_PERCENT=95         # % del saldo disponible a usar por operación
STOP_LOSS_PERCENT=4        # SL fijo desde la entrada (%)
TRAILING_STOP_PERCENT=3    # trailing desde el máximo (%)
MAX_DAILY_LOSS_PERCENT=80  # freno de emergencia: pausa si cae este % del capital inicial
TESTNET=true               # true = testnet | false = producción
```

---

## Uso manual

```bash
./venv/bin/python bot.py
```

Verificá que arranque sin errores y que llegue el mensaje de inicio a Telegram antes de activar el service.

---

## Uso con systemd

```bash
sudo cp trading-bot.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now trading-bot
```

Operación diaria:

```bash
sudo systemctl status trading-bot --no-pager
sudo journalctl -u trading-bot -n 50 --no-pager
sudo systemctl restart trading-bot
```

---

## Estrategia

**Este bot no impone una estrategia fija.** La lógica de entrada y salida va en `bot.py` y los parámetros se controlan desde `.env`.

A modo de ejemplo, una estrategia típica en vela de 1h usando EMA + RSI + MACD:

### Condición de compra (todas deben cumplirse en vela cerrada)

| Condición | Descripción |
|---|---|
| EMA20 cruza arriba EMA50 | Señal de tendencia alcista |
| RSI entre 45 y 70 | Momentum positivo sin sobrecompra |
| MACD > Signal | Confirmación de dirección |

### Condición de venta

| Condición | Descripción |
|---|---|
| Stop Loss fijo | Cierra si el precio cae X% desde la entrada |
| Trailing Stop | Cierra si el precio retrocede X% desde el máximo |
| Cruce bajista | EMA20 cruza abajo EMA50 |

> Adaptá estos indicadores, umbrales y timeframes a tu propio criterio y par elegido.

---

## Gestión de capital y riesgo

| Control | Descripción |
|---|---|
| `CAPITAL_PERCENT` | Porcentaje del saldo disponible a usar por operación |
| `STOP_LOSS_PERCENT` | SL fijo desde el precio de entrada |
| `TRAILING_STOP_PERCENT` | Trailing stop desde el máximo alcanzado |
| `MAX_DAILY_LOSS_PERCENT` | Freno de emergencia: el bot se pausa si el capital cae este % respecto al inicial |

---

## Recuperación de estado

El bot guarda el estado en `state.json` (posición abierta, precio de entrada, máximo para trailing, etc.).

- Si reiniciás el servicio → **recupera el estado y sigue operando**
- Si al iniciar hay saldo en el exchange pero **no existe `state.json`** → el bot avisa por Telegram y **no opera** hasta la próxima señal. Revisá manualmente si necesitás cerrar posición.

---

## Pasar de testnet a producción

En `.env`:

```
TESTNET=false
BINANCE_API_KEY="tu_api_key_real"
BINANCE_API_SECRET="tu_api_secret_real"
```

Recomendaciones de seguridad:
- API configurada **sin permisos de retiro**
- Whitelist de IP del servidor en Binance

---

## Notificaciones Telegram

| Evento | Emoji |
|---|---|
| Bot iniciado | 🤖 |
| Compra ejecutada | 🟢 |
| Venta por Stop Loss | 🔴 |
| Venta por Trailing Stop | 🟡 |
| Venta por señal bajista | 📉 |
| Freno de emergencia activado | ⛔ |
| Error o inconsistencia | ⚠️ |

---

## Archivos del proyecto

| Archivo | Descripción |
|---|---|
| `bot.py` | Lógica principal — estrategia, gestión de capital, ejecución de órdenes |
| `requirements.txt` | Dependencias del proyecto |
| `state.json` | Estado persistente (se crea y actualiza automáticamente) |
| `.env.example` | Template de variables de entorno |
| `trading-bot.service` | Definición del servicio systemd |

---

## Troubleshooting

| # | Síntoma | Causa probable | Solución |
|---|---|---|---|
| 1 | `ModuleNotFoundError` | venv no activado o deps faltantes | `./venv/bin/pip install -r requirements.txt` |
| 2 | `Invalid API key` | `.env` mal configurado o key vencida | Verificar `.env` y permisos en Binance |
| 3 | Bot no responde en Telegram | Token o chat_id incorrecto | Verificar variables en `.env` |
| 4 | systemd status = failed | Error al iniciar | `journalctl -u trading-bot -n 20` |
| 5 | Bot corre pero no opera | Condición de entrada nunca se cumple | Revisar lógica y datos de mercado |
| 6 | Posición abierta que no se cierra | Condición de salida no se evalúa | Revisar logs de cada ciclo y `state.json` |
| 7 | Token no cargado en systemd | Falta `EnvironmentFile=` en el `.service` | Verificar el `.service` |
| 8 | `state.json` inconsistente | Cierre forzado durante una operación | Revisarlo manualmente y corregir o borrar |

---

## Seguridad

- `.env` fuera de git y fuera de backup (`chmod 600 .env`)
- `.gitignore` incluye `venv/`, `.env`, `__pycache__/`, `state.json`, `logs/`
- API de Binance configurada **sin permisos de retiro**
- Whitelist de IP activa en Binance

---

## Próximas mejoras posibles

| Prioridad | Mejora |
|---|---|
| Alta | Trailing stop dinámico por ATR |
| Media | Soporte para múltiples pares simultáneos |
| Media | Dashboard de PnL por Telegram |
| Baja | Backtesting integrado |

---

## Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-03-06 | Versión inicial |

---

## Licencia

MIT
