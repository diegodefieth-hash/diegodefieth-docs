# bot-gastos

Bot personal de Telegram para registrar gastos diarios con mínima fricción. Organiza por categorías configurables, maneja cuotas automáticamente y genera reportes desde Google Sheets.

---

## Requisitos

- Python 3.10+
- Bot de Telegram configurado ([BotFather](https://t.me/BotFather))
- Google Cloud project con Sheets API y Drive API activadas
- Service Account con acceso a la Google Sheet

---

## Instalación

```bash
git clone https://github.com/tu-usuario/bot-gastos
cd bot-gastos

python3 -m venv venv
./venv/bin/python -m pip install -U pip wheel
./venv/bin/pip install -r requirements.txt

cp .env.example .env
nano .env        # completar con tus credenciales
chmod 600 .env
chmod 600 credentials.json
```

### Variables de entorno (`.env.example`)

```
TELEGRAM_BOT_TOKEN="..."
TELEGRAM_CHAT_ID="..."
GOOGLE_CREDENTIALS_PATH="/ruta/al/credentials.json"
GOOGLE_SHEET_ID="..."
```

### Obtener GOOGLE_SHEET_ID

Está en la URL de la Sheet:
```
https://docs.google.com/spreadsheets/d/ESTE_ES_EL_ID/edit
```

### Compartir la Sheet con la Service Account

Abrí el JSON de credenciales, copiá el campo `client_email` y compartí la Sheet con ese email como Editor.

---

## Uso manual

```bash
./venv/bin/python bot.py
```

Verificá que arranque sin errores antes de activar el service.

---

## Uso con systemd

```bash
sudo cp bot-gastos.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now bot-gastos
```

Operación diaria:

```bash
sudo systemctl status bot-gastos --no-pager
sudo journalctl -u bot-gastos -n 50 --no-pager
sudo systemctl restart bot-gastos
```

---

## Comandos del bot

### Registrar un gasto

| Formato | Ejemplo | Comportamiento |
|---|---|---|
| `monto concepto` | `800 café` | Detecta categoría por concepto |
| `monto Nc concepto` | `60000 6c zapatillas` | Cuotas → categoría automática Deudas/Cuotas |

El bot siempre pide confirmación antes de guardar. En la confirmación se puede cambiar la categoría con el botón "📂 Cambiar categoría".

### Consultas

| Comando | Descripción |
|---|---|
| `/hoy` | Gastos del día, desglosados por ítem |
| `/mes` | Resumen del mes por categoría, ordenado por monto |
| `/categorias` | Lista todas las categorías con ejemplos |
| `/ayuda` | Mensaje de ayuda con todos los comandos |

### Editar categorías

| Comando | Ejemplo | Descripción |
|---|---|---|
| `/addcategoria` | `/addcategoria Personales electronica` | Agrega keyword a una categoría |
| `/delcategoria` | `/delcategoria Personales electronica` | Elimina keyword de una categoría |
| `/addcat` | `/addcat Tecnologia` | Crea categoría nueva |
| `/delcat` | `/delcat Tecnologia` | Elimina categoría entera |

---

## Archivos del proyecto

| Archivo | Descripción |
|---|---|
| `bot.py` | Lógica principal del bot |
| `config.json` | Categorías configuradas por el usuario (se crea automáticamente) |
| `requirements.txt` | Dependencias Python |
| `.env.example` | Template de variables de entorno |
| `bot-gastos.service` | Definición del servicio systemd |

> `config.json` se crea automáticamente la primera vez que editás categorías desde Telegram. Hasta entonces el bot usa las categorías por defecto.

---

## Logs y estado

| Comando | Qué muestra |
|---|---|
| `sudo journalctl -u bot-gastos -n 50 --no-pager` | Últimos 50 logs del servicio |
| `sudo journalctl -u bot-gastos -f` | Logs en tiempo real |
| `sudo systemctl status bot-gastos` | Estado del servicio |

Este proyecto no usa carpeta `logs/` — todo va a journalctl.

---

## Categorías por defecto

Cada categoría tiene una lista de keywords. Cuando el concepto del gasto contiene una keyword, se asigna esa categoría automáticamente.

| Categoría | Keywords |
|---|---|
| Hogar | alquiler, expensas, luz, gas, agua, internet, supermercado |
| Personales | ropa, salud, gimnasio, higiene, ocio, electronica, monitor, notebook, celular, tablet |
| Hormiga | café, transporte, kiosco, delivery, uber, taxi |
| Mascotas | alimento, baño, veterinaria, accesorios |
| Hijos | cuota colegio, club, útiles, actividades |
| Deudas/Cuotas | tarjeta, crédito personal |

Reglas automáticas:
- Gasto con cuotas (`Nc`) → siempre **Deudas/Cuotas**
- Sin match de keyword → **Hormiga** por defecto
- En confirmación se puede cambiar con el botón 📂

Las categorías son completamente configurables desde Telegram.

---

## Google Sheets — estructura

Una hoja por mes con nombre `YYYY-MM`. Columnas:

| Fecha | Categoría | Concepto | Monto | Cuota |
|---|---|---|---|---|
| 2026-03-07 | Deudas/Cuotas | monitor | 40000 | 1/6 |

Para editar o borrar gastos: hacerlo directamente en la Sheet. El bot no tiene caché, lee en tiempo real.

**Tipos de gasto:**

| Tipo | Descripción | ¿Suma al total? |
|---|---|---|
| `real` | Gastos normales y pagos de tarjeta | ✅ Sí |
| `informativo` | Cuotas distribuidas en meses futuros | ❌ No |

Las cuotas son `informativo` porque el gasto real se registra cuando pagás la tarjeta. Así no hay doble contabilidad.

---

## Troubleshooting

| # | Síntoma | Causa probable | Solución |
|---|---|---|---|
| 1 | `TypeError: int() argument... NoneType` | `.env` no existe o tiene typo | Verificar que el archivo se llame exactamente `.env` |
| 2 | `Expecting value: line 1 column 1` | `GOOGLE_SHEET_ID` incorrecto o Sheet no compartida | Verificar ID en la URL y que el `client_email` tenga acceso como Editor |
| 3 | Bot no responde en Telegram | Token o chat_id incorrecto | Verificar variables en `.env` |
| 4 | `ModuleNotFoundError` | venv no activado o deps no instaladas | `./venv/bin/pip install -r requirements.txt` |
| 5 | systemd status = failed | Error al iniciar | `journalctl -u bot-gastos -n 20` |
| 6 | Token no cargado en systemd | Falta `EnvironmentFile=` en el `.service` | Verificar el `.service` y hacer `daemon-reload` |
| 7 | Categoría mal detectada | Concepto no matchea ninguna keyword | Usar "📂 Cambiar categoría" en la confirmación, o agregar keyword con `/addcategoria` |
| 8 | `config.json` con datos viejos | Edición manual incorrecta del archivo | Borrar `config.json` para volver a los defaults, o corregir el JSON |

---

## Seguridad

- El bot responde **solo al `CHAT_ID` configurado** — cualquier otro usuario es ignorado
- `.env` y `credentials.json` fuera de git (`chmod 600`, incluidos en `.gitignore`)
- Service Account de Google con acceso **solo a la Sheet necesaria**
- `.gitignore` incluye `venv/`, `.env`, `*.json`, `__pycache__/`

---

## Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-03-07 | v1.0.0 — Versión inicial: registro, cuotas, reportes, categorías por defecto |
| 2026-03-07 | v1.1.0 — Cambio de categoría en confirmación, comandos /addcategoria /delcategoria /addcat /delcat |
| 2026-03-07 | v1.2.0 — Cuotas asignan Deudas/Cuotas automáticamente |
| 2026-03-08 | v1.3.0 — Eliminada subcategoría, /ranking eliminado, /categorias muestra ejemplos |
| 2026-03-08 | v2.0.0 — Tipo real/informativo, mes de inicio de cuotas configurable, /cuotas nuevo comando |

---

## Licencia

MIT
