# bot-noticias

Agente diario que genera un resumen editorial de política argentina y economía mundial usando Claude Haiku (API Anthropic con web search) y lo envía a las 19:00 hs vía Telegram. También guarda el resumen en Supabase para que la web lo muestre.

📢 **Canal de Telegram:** [https://t.me/resumendeldiaarg](https://t.me/resumendeldiaarg)
🌐 **Web pública:** [https://web-noticias-three.vercel.app/](https://web-noticias-three.vercel.app/)

---

## Requisitos

- Python 3.10+
- Dependencias: ver `requirements.txt`
- API key de Anthropic (con acceso a `web_search_20250305`)
- Bot de Telegram ya existente (token + chat_id)

---

## Instalación y setup

```bash
python3 -m venv venv
./venv/bin/pip install -r requirements.txt
cp .env.example .env
chmod 600 .env
# Editar .env con los valores reales
```

### Variables de entorno (`.env.example`)

```
ANTHROPIC_API_KEY="sk-ant-..."
TELEGRAM_BOT_TOKEN="..."
TELEGRAM_CHAT_ID="..."
MAX_TOKENS="8000"

# Supabase — para guardar el resumen en la web
SUPABASE_URL="https://xxxx.supabase.co"
SUPABASE_SERVICE_KEY="eyJ..."   # service_role key (NO la anon)
```

> `MAX_TOKENS`: establecer tras la primera corrida real (tokens consumidos + 35% de margen).

---

## Uso manual

```bash
./venv/bin/python bot.py
```

> Probar manualmente antes de activar el timer. Verificar en Telegram que el resumen llegó y en el dashboard de Anthropic cuántos tokens consumió.

---

## Uso con systemd timer

```bash
# Copiar archivos de servicio
sudo cp bot-noticias.service /etc/systemd/system/
sudo cp bot-noticias.timer /etc/systemd/system/

# Activar
sudo systemctl daemon-reload
sudo systemctl enable --now bot-noticias.timer

# Verificar
sudo systemctl list-timers --all | grep noticias
```

---

## Logs y diagnóstico

| Comando / archivo | Qué muestra |
|---|---|
| `journalctl -u bot-noticias -n 50 --no-pager` | Logs de la última ejecución |
| `tail -f logs/bot_$(date +%Y%m%d).log` | Log del día en tiempo real |
| `systemctl list-timers --all \| grep noticias` | Próxima ejecución programada |
| `systemctl status bot-noticias.timer` | Estado del timer |

---

## Estrategia editorial

| Parámetro | Valor |
|---|---|
| Modelo | `claude-haiku-4-5-20251001` |
| Herramienta | `web_search_20250305` |
| Búsquedas máximas | 12 (`max_uses: 12`) |
| Secciones | Política Argentina + Economía Mundial |
| Noticias por sección | 2–4 según relevancia del día |
| Idioma | Español argentino |
| Horario de envío | 19:00 hs (America/Buenos_Aires) |
| Max tokens | 8000 |
| Costo estimado | ~$0.023/día → ~$0.70/mes |

### Reglas editoriales clave

- Prosa fluida: sin concatenación de frases de distintas fuentes
- **Anti-alucinación:** solo información explícitamente respaldada por los artículos encontrados
- **Atribución:** posiciones de gobierno/sindicatos/oposición siempre atribuidas explícitamente ("Según la CGT...", "El Gobierno sostiene que...")

### Orden de mensajes en Telegram

```
Mensaje 1 → 🗞️ RESUMEN DEL DÍA — DD/MM/YYYY
Mensaje 2 → 🇦🇷 POLÍTICA ARGENTINA
Mensaje 3, 4... → 📰 Noticias de política (una por mensaje)
Mensaje N → 🌍 ECONOMÍA MUNDIAL
Mensaje N+1, N+2... → 📰 Noticias de economía (una por mensaje)
Último mensaje → CIERRE DEL DÍA
```

---

## Notificaciones (Telegram)

Eventos que generan mensaje:
- ✅ Resumen diario completo (ejecución exitosa)
- ❌ Error al llamar a Claude API
- ❌ Variables de entorno faltantes al inicio

Ejemplos:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🗞️ RESUMEN DEL DÍA — 10/03/2026
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🇦🇷 POLÍTICA ARGENTINA
📰 EL GOBIERNO ANUNCIÓ NUEVAS MEDIDAS...
Contexto: ...
Resumen: ...
Análisis: ...

⚠️ bot-noticias: error al generar el resumen.
<ConnectionError: ...>
```

---

## Gestión de riesgo y costos

| Concepto | Valor |
|---|---|
| Tokens estimados por ejecución | 4.000–7.000 (input + output) |
| Costo diario estimado | ~$0.02–$0.05 USD |
| Costo mensual estimado | ~$0.60–$1.50 USD |

**Mitigaciones activas:**
- `MAX_TOKENS` definido en `.env` — limita gasto por llamada
- Una única llamada por ejecución — no hay loops
- Timer oneshot — no hay riesgo de ejecuciones en cadena
- Notificación de error a Telegram si algo falla

---

## Troubleshooting

| # | Síntoma | Causa probable | Solución |
|---|---|---|---|
| 1 | `ModuleNotFoundError` | venv no activado o deps no instaladas | `./venv/bin/pip install -r requirements.txt` |
| 2 | `AuthenticationError` | ANTHROPIC_API_KEY incorrecta o vencida | Verificar `.env` y dashboard de Anthropic |
| 3 | Bot no recibe mensaje en Telegram | TOKEN o CHAT_ID incorrecto | Verificar variables en `.env` |
| 4 | Timer no dispara a las 19:00 | Zona horaria del servidor incorrecta | `timedatectl` — verificar que sea `America/Buenos_Aires` |
| 5 | Ejecución exitosa pero resumen vacío | Bug en extracción de bloques text | Revisar `journalctl` para ver el response completo |
| 6 | `MAX_TOKENS` excedido | Resumen muy largo ese día | Incrementar `MAX_TOKENS` en `.env` + `systemctl daemon-reload` |
| 7 | Timer activo pero nunca ejecuta | `Persistent=true` no disparó al arrancar | `sudo systemctl start bot-noticias.service` para forzar |

---

## Supabase — integración con web-noticias

Al final de cada ejecución exitosa, el bot guarda el resumen en Supabase para que la web lo muestre. Este paso está aislado: si falla, no afecta el envío a Telegram.

### Tabla `resumenes`

| Columna | Tipo | Descripción |
|---|---|---|
| `fecha` | date | Fecha del resumen (ej: 2026-03-17) |
| `tipo` | text | header / seccion_politica / noticia_politica / noticia_economia / cierre |
| `titulo` | text | Título extraído de la noticia (si aplica) |
| `contenido` | text | Texto completo del bloque |
| `orden` | integer | Orden de aparición en el día |

### Comportamiento

- **Anti-duplicados:** si ya existe un resumen para la fecha de hoy, saltea la inserción sin error
- **Fallo aislado:** si Supabase no está disponible, el error se loguea pero el bot termina `OK`
- **Keys usadas:** `SUPABASE_SERVICE_KEY` (service_role) — tiene permisos de INSERT. Nunca usar la anon key en el bot.

### Verificar que guardó

```bash
# En el log del día tiene que aparecer:
grep -i supabase logs/bot_$(date +%Y%m%d).log
# Resultado esperado:
# Supabase: X filas insertadas para 2026-03-17.
# O si ya existía:
# Supabase: ya existe un resumen para 2026-03-17 — saltando inserción.
```

---

## Seguridad

- `.env` fuera de git (`chmod 600 .env`)
- `.gitignore` incluye `.env`, `venv/`, `logs/`
- API key de Anthropic sin permisos innecesarios
- `SUPABASE_SERVICE_KEY` solo en este bot — nunca en la web pública

---

## Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-03-10 | v1.0.0 — Setup inicial |
| 2026-03-10 | v1.0.1 — Cambio a Haiku, max_uses=12, reglas anti-alucinación y atribución, split por noticia, fuentes con links, disable_web_page_preview |
| 2026-03-10 | v1.1.0 — Nuevo formato estructurado (HECHOS CONFIRMADOS, POR QUÉ IMPORTA, QUÉ FALTA CONFIRMAR, LECTURA EDITORIAL, ESTADO). Separador === NOTICIA ===. Cierre del día agregado. |
| 2026-03-11 | v1.2.0 — Secciones separadas (POLÍTICA / ECONOMÍA), titular con 📰, instrucciones anti-dramatismo en cierre del día. |
| 2026-03-12 | v1.2.1 — Agregadas reglas anti-noticias-viejas: fecha explícita en USER_PROMPT y restricción de 24hs en SYSTEM_PROMPT. |
| 2026-03-17 | v1.3.1 — Integración Supabase: `guardar_en_supabase()` al final de `main()`, anti-duplicados por fecha, `supabase>=2.0.0` en requirements. |
| 2026-03-20 | v1.4.0 — fix crítico: REGLA CRÍTICA en system prompt, Claude nunca rechaza la generación. Mínimo noticias bajado de 5 a 2 |
