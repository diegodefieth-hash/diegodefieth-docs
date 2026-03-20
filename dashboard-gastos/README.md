# dashboard-gastos

Panel web personal para visualizar los gastos registrados por `bot-gastos`. Lee directamente de Google Sheets sin base de datos ni backend propio.
## 🌐 Ver demo

👉 **[diegodefieth-hash.github.io/diegodefieth-docs/dashboard-gastos](https://diegodefieth-hash.github.io/diegodefieth-docs/dashboard-gastos/)**

> Demo con datos ficticios — el dashboard real conecta a tu Google Sheet.


> Diseñado para correr en un servidor local accesible por red privada (Tailscale, VPN, LAN).

---

## Requisitos

- Python 3.10+
- Google Cloud Service Account con acceso a la Sheet de `bot-gastos`
- Servidor Linux con systemd

---

## Instalación

```bash
git clone https://github.com/tu-usuario/dashboard-gastos
cd dashboard-gastos

python3 -m venv venv
./venv/bin/python -m pip install -U pip wheel
./venv/bin/pip install -r requirements.txt

cp .env.example .env
nano .env        # completar con tus credenciales
chmod 600 .env
```

### Variables de entorno (`.env.example`)

```
GOOGLE_CREDENTIALS_PATH="/ruta/al/credentials.json"
GOOGLE_SHEET_ID="..."
FLASK_HOST="0.0.0.0"
FLASK_PORT=8003
```

---

## Uso manual

```bash
./venv/bin/python app.py
```

Verificá que arranque sin errores antes de activar el service.

---

## Uso con systemd

```bash
sudo cp dashboard-gastos.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now dashboard-gastos
```

Operación diaria:

```bash
sudo systemctl status dashboard-gastos --no-pager
sudo journalctl -u dashboard-gastos -n 50 --no-pager
sudo systemctl restart dashboard-gastos
```

---

## Vistas del panel

- **Resumen** — KPIs (total real, cuotas, categorías activas, gasto diario promedio), torta de distribución, lista de categorías con barra proporcional
- **Detalle** — Tabla completa de gastos reales del mes, ordenable por columna
- **Cuotas** — Total comprometido en cuotas + tabla con progreso (ej: 2/6)
- **Comparar** — Selector de dos meses, delta porcentual, gráfico de barras agrupado por categoría

Al cargar, selecciona automáticamente el mes en curso (o el más cercano anterior si no existe).

---

## Endpoints API

| Endpoint | Parámetros | Retorna |
|---|---|---|
| `GET /` | — | Panel HTML |
| `GET /api/meses` | — | Lista de meses disponibles |
| `GET /api/gastos` | `?mes=YYYY-MM` | Total real, por categoría, cuotas informativas |
| `GET /api/items` | `?mes=YYYY-MM` | Listado completo de gastos reales |
| `GET /api/comparar` | `?mes1=YYYY-MM&mes2=YYYY-MM` | Datos de ambos meses para comparación |

> Los datos se leen con `value_render_option='UNFORMATTED_VALUE'` para manejar correctamente el formato numérico argentino (punto como separador de miles, coma como decimal).
> Caché en memoria de 120 segundos por mes. Se invalida al reiniciar el servicio.

---

## Archivos del proyecto

| Archivo | Descripción |
|---|---|
| `app.py` | Servidor Flask, endpoints API, caché en memoria 120s |
| `templates/index.html` | SPA del panel (HTML + CSS + JS en un solo archivo) |
| `requirements.txt` | Dependencias Python |
| `.env.example` | Template de variables de entorno |
| `dashboard-gastos.service` | Definición del servicio systemd |

---

## Relación con bot-gastos

Proyecto de **solo lectura**. No escribe en la Sheet, no comparte proceso ni puerto con el bot. Son servicios independientes que usan la misma Google Sheet como fuente de datos.

---

## Troubleshooting

| # | Síntoma | Causa probable | Solución |
|---|---|---|---|
| 1 | `ModuleNotFoundError` | venv no activado o deps faltantes | `./venv/bin/pip install -r requirements.txt` |
| 2 | `TemplateNotFound: index.html` | Falta carpeta `templates/` o el archivo | Verificar que existe `templates/index.html` |
| 3 | `FileNotFoundError` en credenciales | Ruta del JSON incorrecta en `.env` | Verificar `GOOGLE_CREDENTIALS_PATH` |
| 4 | `Worksheet not found` | La hoja no existe o el nombre no es `YYYY-MM` | Verificar nombre exacto de la hoja en la Sheet |
| 5 | Panel carga pero sin datos | `GOOGLE_SHEET_ID` incorrecto o sin permisos | Verificar ID y que el `client_email` tenga acceso como Editor |
| 6 | Montos incorrectos (multiplicados x100) | `get_all_records()` sin `UNFORMATTED_VALUE` | Verificar que el código usa `value_render_option='UNFORMATTED_VALUE'` |
| 7 | systemd status = failed | Error al iniciar Flask | `journalctl -u dashboard-gastos -n 20` |
| 8 | Panel no accesible desde otro dispositivo | Flask escucha en 127.0.0.1 | Verificar `FLASK_HOST=0.0.0.0` en `.env` |
| 9 | Datos desactualizados | Caché activo (120s) | Reiniciar el servicio para limpiar caché |
| 10 | Cambio en `index.html` no se ve | Flask cachea templates en memoria | `sudo systemctl restart dashboard-gastos` |

---

## Seguridad

- El servidor **no debe estar expuesto en el router** — solo accesible por red privada (Tailscale, VPN, LAN)
- Verificar con `sudo ss -tlnp | grep 8003` que Flask escucha en la IP correcta
- `.env` fuera de git (`chmod 600 .env`)
- `.gitignore` incluye `venv/`, `.env`, `*.json`, `__pycache__/`
- Nunca usar `debug=True` en el servicio systemd

---

## Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-03-09 | v1.0.0 — Implementación inicial: resumen, detalle, cuotas, comparación |
| 2026-03-09 | Bugfix: montos con formato argentino → `value_render_option='UNFORMATTED_VALUE'` |
| 2026-03-09 | Rediseño visual, tipografía de sistema |
| 2026-03-09 | Fix: selector de mes arranca en el mes en curso |
| 2026-03-09 | Servicio systemd instalado y activo |
| 2026-03-09 | v1.0.1 — Tabla Detalle ordenable por columna con clic en header, indicador ↑↓ |

---

## Licencia

MIT
