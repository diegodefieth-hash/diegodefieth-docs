# Plantilla: README para bots de home server

Plantilla para documentar bots y herramientas que corren en un servidor personal (Linux/Lubuntu). Cubre todo el ciclo: instalación, configuración, operación diaria y troubleshooting.

Está pensada para que cualquiera pueda armar su propia versión de un bot a partir de la documentación, sin necesidad de ver el código fuente.

---

## Cómo usar esta plantilla

1. Copiá el contenido de `TEMPLATE.md`
2. Reemplazá cada sección con la información de tu proyecto
3. Borrá las secciones que no apliquen (por ejemplo, "Uso con cron" si usás systemd, o viceversa)
4. Completá el `.env.example` con todas las variables que usa tu bot

---

## Estructura del README generado

```
# nombre-del-bot
Descripción breve

## Requisitos
## Instalación
## Variables de entorno
## Uso manual
## Uso con systemd  ← o cron, según el caso
## Comandos del bot  ← si aplica (bots de Telegram)
## Lógica principal  ← filtros, estrategia, parámetros
## Controles implementados
## Ejemplos de output
## Archivos del proyecto
## Troubleshooting
## Seguridad
## Próximas mejoras
## Historial de cambios
```

---

## Convenciones que uso

- **systemd** para servicios permanentes — `Type=simple`, `Restart=on-failure`
- **cron** para tareas puntuales o periódicas simples
- **venv** siempre — nunca `/usr/bin/python3` directamente en services
- **`.env`** para todas las credenciales — nunca en el código, nunca en git
- **journalctl** como destino de logs para servicios systemd — sin carpeta `logs/` por defecto
- Todo va a producción directo — sin staging

---

## Template

````markdown
# nombre-del-bot

Descripción breve: qué hace, qué problema resuelve, qué APIs usa.

> **Fuente de datos:** [Nombre del servicio](https://url) — [requiere API key / es pública]

---

## Requisitos

- Python 3.10+
- [Servicio externo que usa] ([link])
- Bot de Telegram configurado ([BotFather](https://t.me/BotFather)) — si aplica
- API key de [servicio] — si aplica

---

## Instalación

```bash
git clone https://github.com/tu-usuario/nombre-del-bot
cd nombre-del-bot

python3 -m venv venv
./venv/bin/python -m pip install -U pip wheel
./venv/bin/pip install -r requirements.txt

cp .env.example .env
nano .env        # completar con tus credenciales
chmod 600 .env
```

### Variables de entorno (`.env.example`)

```
TELEGRAM_BOT_TOKEN="..."
TELEGRAM_CHAT_ID="..."
# Agregar las variables que use tu bot
```

---

## Uso manual

```bash
./venv/bin/python bot.py
```

Verificá que arranque sin errores antes de activar el service.

---

## Uso con systemd

```bash
sudo cp nombre-del-bot.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now nombre-del-bot
```

Operación diaria:

```bash
sudo systemctl status nombre-del-bot --no-pager
sudo journalctl -u nombre-del-bot -n 50 --no-pager
sudo systemctl restart nombre-del-bot
```

> Si no usás systemd, reemplazá esta sección con la configuración de cron:
> ```
> */30 * * * * /ruta/al/venv/bin/python /ruta/al/bot.py >> /ruta/logs/cron.log 2>&1
> ```

---

## Comandos del bot  ← borrar si no es un bot de Telegram

| Comando | Descripción |
|---|---|
| `/comando1` | Descripción |
| `/comando2` | Descripción |

---

## Lógica principal

| Parámetro | Valor |
|---|---|
| Parámetro 1 | Valor actual |
| Parámetro 2 | Valor actual |

### Incluido

- Item 1
- Item 2

### Excluido deliberadamente

- Item 1 — motivo
- Item 2 — motivo

---

## Controles implementados

| Control | Estado | Valor/Detalle |
|---|---|---|
| Control 1 | ✅ | Descripción |
| Control 2 | ✅ | Descripción |
| Control pendiente | ❌ | No implementado |

---

## Ejemplos de output

```
Ejemplo de mensaje o resultado real
```

---

## Archivos del proyecto

| Archivo | Descripción |
|---|---|
| `bot.py` | Lógica principal |
| `requirements.txt` | Dependencias |
| `.env.example` | Template de variables |
| `nombre.service` | Definición del servicio systemd |

---

## Troubleshooting

| # | Síntoma | Causa probable | Solución |
|---|---|---|---|
| 1 | `ModuleNotFoundError` | venv no activado o deps faltantes | `./venv/bin/pip install -r requirements.txt` |
| 2 | Bot no responde | Token incorrecto | Verificar `.env` y que el service esté activo |
| 3 | systemd status = failed | Error al iniciar | `journalctl -u nombre-del-bot -n 20` |
| 4 | Token no cargado en systemd | Falta `EnvironmentFile=` en el `.service` | Verificar el `.service` |

---

## Seguridad

- `.env` fuera de git y fuera de backup (`chmod 600 .env`)
- `.gitignore` incluye `venv/`, `.env`, `__pycache__/`
- [Agregar notas específicas de seguridad del proyecto]

---

## Próximas mejoras

| Prioridad | Mejora |
|---|---|
| Alta | ... |
| Media | ... |
| Baja | ... |

---

## Historial de cambios

| Fecha | Cambio |
|---|---|
| YYYY-MM-DD | Versión inicial |

---

## Licencia

MIT
````

---

## Notas sobre el `.service` de systemd

Template base para un bot permanente:

```ini
[Unit]
Description=Nombre del bot
After=network.target

[Service]
Type=simple
User=tu-usuario
WorkingDirectory=/ruta/al/bot
EnvironmentFile=/ruta/al/bot/.env
ExecStart=/ruta/al/bot/venv/bin/python bot.py
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Puntos clave:
- `EnvironmentFile=` apunta al `.env` — así las credenciales no van en el `.service`
- `ExecStart` usa el python del venv, no `/usr/bin/python3`
- `WorkingDirectory` tiene que ser la ruta absoluta al proyecto
- Después de cualquier cambio en el `.env`: `sudo systemctl daemon-reload`

---

## Licencia

MIT
