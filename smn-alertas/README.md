# smn-alertas

Monitor de alertas meteorológicas del [SMN (Servicio Meteorológico Nacional)](https://www.smn.gob.ar) para una zona configurable. Manda email automático cuando cambia el estado de alerta — aparece una nueva o se levanta. No usa Telegram ni bot, solo email vía Gmail SMTP.

---

## Requisitos

- Python 3.10+
- Dependencias: `curl_cffi`, `requests`, `python-dotenv`
- Cuenta de Gmail con verificación en dos pasos activada (para App Password)
- No requiere venv — se puede instalar con `pip3 install --user`

---

## Instalación

```bash
git clone https://github.com/tu-usuario/smn-alertas
cd smn-alertas

pip3 install curl_cffi requests python-dotenv

cp .env.example .env
nano .env          # completar con tus credenciales
chmod 600 .env
```

### Variables de entorno (`.env.example`)

```
GMAIL_USER="tu@gmail.com"
GMAIL_PASSWORD="xxxxxxxxxxxxxxxx"
DEST_EMAIL="destino@gmail.com"
```

> `GMAIL_PASSWORD` es una App Password de Google (sin espacios).
> Generarla en: [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords) — requiere 2FA activado.
> `GMAIL_USER` y `DEST_EMAIL` pueden ser el mismo Gmail.

---

## Uso manual

```bash
python3 check_alertas_v2.py
```

Si termina con `Sin cambios.` o `Email enviado.` → está funcionando correctamente.

> Siempre probar manualmente antes de activar el cron.

---

## Uso con cron

Este proyecto corre vía cron. No tiene `.service` de systemd configurado.

```bash
crontab -e
```

```
0 * * * * python3 /ruta/a/smn-alertas/check_alertas_v2.py >> /ruta/a/smn-alertas/alertas.log 2>&1
```

---

## Logs y estado

| Archivo / comando | Qué contiene |
|---|---|
| `alertas.log` | Output completo de cada ejecución del cron |
| `last_state.json` | Estado anterior: `hay_alerta` (bool) + timestamp |
| `token_cache.json` | JWT cacheado del SMN, se renueva automáticamente |
| `crontab -l \| grep smn` | Verificar que el cron está activo |
| `tail -30 alertas.log` | Ver las últimas ejecuciones |

---

## Cómo funciona

### Arquitectura

- **Tipo:** script puntual vía cron (no service systemd)
- **Trigger:** cron cada hora (`0 * * * *`)
- **Fuente:** API oficial `ws1.smn.gob.ar` con JWT extraído de `smn.gob.ar` via `curl_cffi`
- **Notificación:** email vía Gmail SMTP SSL puerto 465 + App Password
- **Estado:** persistido en `last_state.json` entre ejecuciones

### Por qué curl_cffi

`smn.gob.ar` está protegido por Cloudflare y bloquea con 403 cualquier cliente sin fingerprint TLS de navegador real. `curl_cffi` con `impersonate="chrome"` resuelve esto. La API `ws1.smn.gob.ar` acepta el JWT extraído del HTML con header `Authorization: JWT <token>` — importante: no es `Bearer`, es `JWT`.

### Flujo de ejecución

```
cron (cada hora)
       │
       ▼
check_alertas_v2.py
       │
       ▼
curl_cffi GET smn.gob.ar → extraer JWT → cachear en token_cache.json
       │
       ▼
GET ws1.smn.gob.ar/v1/warning/alert/area/{AREA_ID}  (Authorization: JWT <token>)
       │
       ▼
¿eventos con max_level > 1?
       │
       ├── ¿Cambió vs last_state.json?
       │         │
       │        SÍ → send_email() → guardar nuevo estado
       │         │
       │        NO → "Sin cambios." → fin
       ▼
Escribir en alertas.log
```

---

## Niveles de alerta SMN

| Nivel | Color | Significado |
|---|---|---|
| 1 | 🟢 Verde | Sin alerta |
| 2 | 🟣 Violeta | Advertencia |
| 3 | 🟡 Amarillo | Alerta amarillo |
| 4 | 🟠 Naranja | Alerta naranja |
| 5 | 🔴 Rojo | Alerta rojo |

Solo se notifica cuando `max_level > 1` en el área configurada.

---

## Ejemplos de email

```
Asunto: Alerta SMN activa

Alertas SMN al 21/03/2026 21:00:

  🟡 Tormenta — alerta amarillo
     Tormentas fuertes con granizo

https://www.smn.gob.ar/alertas_area?loc=762
```

```
Asunto: Alertas SMN levantadas

Alertas SMN al 21/03/2026 23:00:

  sin alertas

https://www.smn.gob.ar/alertas_area?loc=762
```

---

## Archivos del proyecto

| Archivo | Descripción |
|---|---|
| `check_alertas_v2.py` | Script principal — obtiene JWT, consulta API, compara estado, envía email |
| `last_state.json` | Estado persistente entre ejecuciones (se crea automáticamente) |
| `token_cache.json` | JWT cacheado del SMN (se crea automáticamente) |
| `alertas.log` | Log de cada ejecución del cron |
| `.env.example` | Template de variables de entorno |

---

## Troubleshooting

| # | Síntoma | Causa probable | Solución |
|---|---|---|---|
| 1 | `RuntimeError: JWT no encontrado` | smn.gob.ar cambió el HTML | Actualizar regex en `get_jwt()` |
| 2 | `401 Unauthorized` en ws1.smn.gob.ar | JWT expirado o inválido | `rm token_cache.json` y re-ejecutar |
| 3 | `SMTPAuthenticationError` | App Password incorrecta o expirada | Regenerar en myaccount.google.com/apppasswords |
| 4 | Email no llega | Filtros de spam | Revisar carpeta spam |
| 5 | Cron no dispara | Error en crontab o cron detenido | `crontab -l`, `sudo systemctl status cron` |
| 6 | Script corre sin errores pero no manda email | Estado no cambió | Normal — `rm last_state.json` para forzar envío |
| 7 | `403 Forbidden` al obtener JWT | Cloudflare actualizó detección | Actualizar `curl_cffi` o cambiar target de `impersonate` |

> El caso 6 es el más frecuente: el proceso termina sin errores pero no hace nada. Es el comportamiento correcto cuando no hay cambio de estado. Verificar `last_state.json` antes de asumir que está roto.

---

## Checklist "está vivo"

- [ ] `crontab -l | grep smn` → 1 entrada, cada hora
- [ ] `tail -5 alertas.log` → última ejecución reciente, sin errores
- [ ] `last_state.json` existe y tiene `hay_alerta`
- [ ] Test manual → termina con `Sin cambios.` o `Email enviado.`

---

## Plan de emergencia

Si la API del SMN cambia estructura o el JWT deja de funcionar, revisar cómo el sitio oficial consume la API:

```
smn.gob.ar/sites/all/modules/custom/smn_alertas/js/matriz_alertas_area.js
```

Este JS contiene las llamadas ajax con los headers y endpoints exactos que usa el frontend oficial.

---

## Seguridad

- `.env` con `chmod 600`, fuera de git y fuera de backup
- App Password de Gmail — no requiere acceso completo a la cuenta
- `.gitignore` incluye: `.env`, `token_cache.json`, `last_state.json`, `alertas.log`

---

## Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-03-21 | v1: scraping HTML con `requests` — fallaba con Cloudflare 403 |
| 2026-03-21 | v2: `curl_cffi` + API JWT `ws1.smn.gob.ar` + header `JWT` (no `Bearer`) |

---

## Licencia

MIT
