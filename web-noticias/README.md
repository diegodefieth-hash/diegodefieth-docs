# web-noticias

Web pública de noticias generadas íntegramente por IA — resumen diario automatizado de política argentina y economía mundial.

🌐 **[https://web-noticias-three.vercel.app/](https://web-noticias-three.vercel.app/)**

---

## Objetivo

**¿Qué es esto?**
Un experimento personal para demostrar que hoy es posible construir una página de noticias completamente automatizada usando inteligencia artificial, sin intervención humana en la generación del contenido.

**¿Cómo funciona?**
Todos los días a las 19:00 hs un agente de IA analiza fuentes periodísticas de Argentina y el mundo, elige las noticias más relevantes, las procesa y redacta su propia versión. El proceso es completamente autónomo. Ninguna persona elige las noticias, redacta los textos ni revisa el contenido antes de que se publique.

**¿De dónde vienen las noticias?**
Cada nota incluye el link a la fuente original. El agente consulta Reuters, AP, BBC, Bloomberg, La Nación, Infobae y Financial Times, entre otros. Se recomienda verificar en la fuente antes de compartir.

**¿Quién es responsable del contenido?**
El contenido es generado íntegramente por un agente de IA. Las opiniones y lecturas editoriales son del agente, no de una persona. Este sitio no tiene redacción ni editor.

**¿Por qué?**
Para explorar los límites de la IA aplicada al periodismo. Este proyecto no busca reemplazar al periodismo humano sino entender hasta dónde puede llegar una IA trabajando sola.

---

## Requisitos

- Python 3.10+
- Dependencias: ver `requirements.txt`
- Cuenta en Supabase con tabla `resumenes` creada
- bot-noticias corriendo y escribiendo en Supabase

---

## Instalación y setup

```bash

python3 -m venv venv
./venv/bin/pip install -r requirements.txt
cp .env.example .env
chmod 600 .env
# Editar .env con los valores reales
# Copiar banner: cp /ruta/banner.jpg static/banner.jpg
```

### Variables de entorno (`.env.example`)

```
FLASK_HOST="0.0.0.0"
FLASK_PORT="8005"
FLASK_DEBUG="false"

# Supabase — anon public key (solo lectura, nunca la service_role)
SUPABASE_URL="https://xxxx.supabase.co"
SUPABASE_KEY="eyJ..."
```

---

## Uso manual

```bash
./venv/bin/python app.py
```

> Probar manualmente antes de activar systemd.

---

## Uso con systemd

```bash
sudo cp web-noticias.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now web-noticias
```

---

## Estructura del proyecto

```
web-noticias/
├── app.py                  # Flask app — lee de Supabase
├── templates/
│   └── index.html          # Template Jinja2 con modo claro/oscuro
├── static/
│   └── banner.jpg          # Banner del header (1200×200 px)
├── .env                    # Variables de entorno (no en git)
├── .env.example            # Plantilla de variables
├── .gitignore
├── requirements.txt
├── VERSION
├── web-noticias.service    # Servicio systemd
└── README.md
```

---

## Arquitectura

```
bot-noticias (server casero)
    ↓ 1. envía resumen a Telegram
    ↓ 2. guarda en Supabase (tabla: resumenes)

Supabase (PostgreSQL en la nube)
    ↓ la web lee desde ahí con anon key (solo lectura)

web-noticias (Flask)
    ├── Local: systemd — red local / VPN
    └── Público: Vercel — HTTPS, sin sleep
```

---

## Supabase — tabla `resumenes`

| Columna | Tipo | Descripción |
|---|---|---|
| `id` | bigint autoincrement | Clave primaria |
| `fecha` | date | Fecha del resumen (ej: 2026-03-17) |
| `tipo` | text | header / seccion_politica / noticia_politica / seccion_economia / noticia_economia / cierre |
| `titulo` | text | Título de la noticia (si aplica) |
| `contenido` | text | Texto completo del bloque |
| `orden` | integer | Orden de aparición en el día |
| `created_at` | timestamptz | Fecha de inserción (automática) |

### Seguridad Supabase

- **anon key** (web): solo puede hacer SELECT — nunca escribe
- **service_role key** (bot.py): tiene permisos de INSERT — nunca va en la web
- RLS habilitado con policy `lectura publica` (SELECT para todos)

---

## Logs y diagnóstico

| Comando | Qué muestra |
|---|---|
| `sudo systemctl status web-noticias` | Estado del servicio |
| `sudo journalctl -u web-noticias -n 50 --no-pager` | Logs recientes |
| `sudo journalctl -u web-noticias -f` | Logs en tiempo real |
| `ss -tlnp \| grep 8005` | Verifica que el puerto está escuchando |

---

## Fases de implementación

| Fase | Estado | Descripción |
|---|---|---|
| Fase 2 | ✅ Completa | Datos hardcodeados para validación visual |
| Fase 3 | ✅ Completa | Supabase conectado: bot escribe, web lee |
| Fase 3b | ✅ Completa | systemd service activado (Restart=always) |
| Fase 4 | ✅ Completa | Deploy en Vercel — https://web-noticias-three.vercel.app/ |

---

## Troubleshooting

| # | Síntoma | Causa probable | Solución |
|---|---|---|---|
| 1 | `ModuleNotFoundError` | venv no activado | `./venv/bin/pip install -r requirements.txt` |
| 2 | Puerto 8005 no responde | Servicio no corriendo | `sudo systemctl start web-noticias` |
| 3 | Banner no carga | Falta `static/banner.jpg` | Copiar imagen al directorio `static/` |
| 4 | Cambios en `index.html` no aparecen | Flask cachea templates | `sudo systemctl restart web-noticias` |
| 5 | `.env` no carga en systemd | Falta `daemon-reload` | `sudo systemctl daemon-reload` tras editar `.env` |
| 6 | Web sin noticias | Supabase vacío o key incorrecta | Verificar SUPABASE_URL y SUPABASE_KEY en `.env` |
| 7 | Web muestra datos viejos | systemd tiene `.env` cacheado | `sudo systemctl daemon-reload && sudo systemctl restart web-noticias` |

---

## Seguridad

- `.env` fuera de git (`chmod 600 .env`)
- `.gitignore` incluye `.env`, `venv/`, `__pycache__/`
- `FLASK_DEBUG=false` en producción
- Supabase anon key en la web (solo lectura)
- Supabase service_role key solo en bot-noticias (nunca en la web)

---

## Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-03-17 | v1.0.0 — Setup inicial Fase 2: datos hardcodeados, modo claro/oscuro, banner |
| 2026-03-17 | v1.0.1 — Banner: `height: auto` para evitar recorte de imagen 1200×200 |
| 2026-03-17 | v1.1.0 — Fase 3: `app.py` conectado a Supabase (anon key, solo lectura) |
| 2026-03-17 | v1.1.1 — Fase 3b: systemd service activado en producción (Restart=always) |
| 2026-03-17 | v1.2.0 — Nuevo diseño: encabezado oscuro, secciones en píldoras, tarjetas con chevron, badges de estado con 3 colores, cierre con borde de acento. Backup del diseño anterior en `index.html.bak` |
| 2026-03-17 | v1.2.1 — Nueva página `/about` (`about.html`) con texto "Acerca de este proyecto". Link al pie de `index.html`. Ruta `/about` en `app.py` |
| 2026-03-18 | v1.3.0 — Múltiples fuentes por noticia: `_extraer_fuentes()` con regex. Template itera lista de fuentes con flex-wrap |
| 2026-03-20 | v2.0.0 — Fase 4 completa: deploy en Vercel con `vercel.json` y gunicorn. URL: https://web-noticias-three.vercel.app/ |
