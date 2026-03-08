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
chmod 600 *.json # credenciales de Google
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
| `/ranking` | Igual que /mes |
| `/categorias` | Lista todas las categorías y sus subcategorías |
| `/ayuda` | Mensaje de ayuda con todos los comandos |

### Editar categorías

| Comando | Ejemplo | Descripción |
|---|---|---|
| `/addcategoria` | `/addcategoria Personales electronica` | Agrega subcategoría |
| `/delcategoria` | `/delcategoria Personales electronica` | Elimina subcategoría |
| `/addcat` | `/addcat Tecnologia` | Crea categoría nueva |
| `/delcat` | `/delcat Tecnologia` | Elimina categoría entera |

---

## Categorías por defecto

| Categoría | Subcategorías |
|---|---|
| Hogar | alquiler, expensas, luz, gas, agua, internet, supermercado |
| Personales | ropa, salud, gimnasio, higiene, ocio, electronica, monitor, notebook, celular, tablet |
| Hormiga | café, transporte, kiosco, delivery, uber, taxi |
| Mascotas | alimento, baño, veterinaria, accesorios |
| Hijos | cuota colegio, club, útiles, actividades |
| Deudas/Cuotas | tarjeta, crédito personal |

Reglas automáticas:
- Cualquier gasto con cuotas (`Nc`) se asigna a **Deudas/Cuotas** sin importar el concepto
- Si ninguna subcategoría matchea → **Hormiga** por defecto

Las categorías son completamente configurables desde Telegram con los comandos `/addcat`, `/delcat`, `/addcategoria`, `/delcategoria`.

---

## Google Sheets — estructura

Una hoja por mes con nombre `YYYY-MM`. Columnas:

| Fecha | Categoría | Subcategoría | Concepto | Monto | Cuota |
|---|---|---|---|---|---|
| 2026-03-07 | Personales | electronica | monitor | 40000 | 1/6 |

Para editar o borrar gastos: hacerlo directamente en la Sheet. El bot no tiene caché, lee en tiempo real.

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

## Troubleshooting

| # | Síntoma | Causa probable | Solución |
|---|---|---|---|
| 1 | `TypeError: int() argument... NoneType` | `.env` no existe o tiene typo en el nombre | Verificar que el archivo se llame exactamente `.env` |
| 2 | `Expecting value: line 1 column 1` | `GOOGLE_SHEET_ID` incorrecto o Sheet no compartida | Verificar ID en la URL y que el `client_email` tenga acceso como Editor |
| 3 | Bot no responde en Telegram | Token o chat_id incorrecto | Verificar variables en `.env` |
| 4 | `ModuleNotFoundError` | venv no activado o deps no instaladas | `./venv/bin/pip install -r requirements.txt` |
| 5 | systemd status = failed | Error al iniciar | `journalctl -u bot-gastos -n 20` |
| 6 | Token no cargado en systemd | Falta `EnvironmentFile=` en el `.service` | Verificar el `.service` y hacer `daemon-reload` |
| 7 | Categoría mal detectada | Concepto no matchea ninguna subcategoría | Usar "📂 Cambiar categoría" en la confirmación, o agregar subcategoría con `/addcategoria` |
| 8 | `config.json` con datos viejos | Edición manual incorrecta del archivo | Borrar `config.json` para volver a los defaults, o corregir el JSON |

---

## Seguridad

- El bot responde **solo al `CHAT_ID` configurado** — cualquier otro usuario es ignorado
- `.env` y `*.json` (credenciales Google) fuera de git (`chmod 600`, incluidos en `.gitignore`)
- Service Account de Google con acceso **solo a la Sheet necesaria**
- `.gitignore` incluye `venv/`, `.env`, `*.json`, `__pycache__/`

---

## Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-03-07 | v1.0.0 — Versión inicial: registro, cuotas, reportes, categorías por defecto |
| 2026-03-07 | v1.1.0 — Cambio de categoría en confirmación, comandos /addcategoria /delcategoria /addcat /delcat |
| 2026-03-07 | v1.2.0 — Cuotas asignan Deudas/Cuotas automáticamente |

---

## Licencia

MIT
