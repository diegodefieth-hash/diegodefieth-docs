# defi-stable-bot

Bot de Telegram que consulta [DeFiLlama](https://defillama.com) en tiempo real y reporta oportunidades de depósito en stablecoins en redes Base y Arbitrum. Solo muestra protocolos de depósito de un único activo (lending/yield). Sin pools de liquidez, sin pares volátil-stable.

> **Fuente de datos:** API pública de DeFiLlama — no requiere API key.

---

## Funcionalidades

- `/oportunidades` — Top 10 oportunidades en Base y Arbitrum
- `/base` / `/arbitrum` — Filtrar por red
- `/analizar <red> <token> <protocolo>` — Análisis de riesgo con IA y búsqueda web en tiempo real
- `/filtros` — Panel interactivo para ajustar umbrales de APY, TVL y tipo de rendimiento
- `/auto` / `/stopauto` — Reportes automáticos a las 8am y 8pm
- `/ayuda` — Ayuda y guía de lectura del reporte

---

## Requisitos

- Python 3.10+
- Token de bot de Telegram ([BotFather](https://t.me/BotFather))
- API key de Anthropic (solo necesaria para `/analizar`)
- Acceso a internet (DeFiLlama es pública, sin API key)

---

## Instalación

```bash
git clone https://github.com/tu-usuario/defi-stable-bot
cd defi-stable-bot

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
ANTHROPIC_API_KEY="..."   # opcional — solo necesaria para /analizar
```

---

## Uso manual

```bash
./venv/bin/python bot.py
```

Si aparece `Bot DeFi iniciado.` y `getMe 200 OK` el bot está funcionando correctamente.

---

## Uso con systemd

```bash
sudo cp defi-stable-bot.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now defi-stable-bot
```

Operación diaria:

```bash
sudo systemctl status defi-stable-bot --no-pager
sudo journalctl -u defi-stable-bot -n 50 --no-pager
sudo systemctl restart defi-stable-bot
```

---

## Comandos del bot

| Comando | Descripción |
|---|---|
| `/oportunidades` | Top 10 por red — Base y Arbitrum en mensajes separados |
| `/base` | Solo Base |
| `/arbitrum` | Solo Arbitrum |
| `/analizar <red> <token> <protocolo>` | Análisis de riesgo con IA y búsqueda web |
| `/filtros` | Panel interactivo de filtros (APY, TVL, red, tipo de yield) |
| `/auto` | Reporte automático cada 12h (8am y 8pm) |
| `/stopauto` | Detener reporte automático |
| `/ayuda` | Ayuda |

### Ejemplos de `/analizar`

```
/analizar base USDC aave-v3
/analizar arbitrum USDC gains-network
/analizar base USDC wasabi
```

---

## Lógica de filtrado

| Parámetro | Valor |
|---|---|
| Redes | Base, Arbitrum |
| Tipo de depósito | Un solo activo stable (lending/yield) |
| TVL mínimo | $500,000 |
| APY mínimo | 3% |
| APY máximo | 500% (filtro anti-outlier) |
| Flag de cambio brusco | >50% en 24h |
| Resultados por red | Top 10 ordenados por APY desc |

### Protocolos incluidos (whitelist)

**Lending clásico:** Aave v2/v3, Compound v2/v3, Morpho, Spark, Euler v2.  
**Lending en Base/Arbitrum:** Fluid Lending, Moonwell, Seamless, Silo v2, Radiant, Dolomite, Deltaprime, Arcadia, Extra Finance (xlend), Exactly, Prime Protocol, Tender.  
**Yield vaults de un activo:** Yearn, Harvest, Lazy Summer, Fusion by IPOR, Autofinance, Yo Protocol, Sprinter, MaxAPY, Lagoon.  
**Perps con USDC como colateral:** Gains Network, WooFi Earn, Avantis, KiloEx, Harmonix, Wasabi, MortgageFi.  
**Otros:** Curve LlamaLend, Overnight Finance, Stake DAO, Superfund, TermMax, Goat Protocol, Snowbl Capital.

### Excluidos deliberadamente

- **Merkl** — distribuidor de rewards, no protocolo de depósito
- **Beefy** — vault de LP, no depósito de un solo activo
- Pools de liquidez (stable-stable y volátil-stable)
- Cualquier símbolo con más de un token

---

## Controles de riesgo

| Control | Estado | Valor |
|---|---|---|
| Whitelist de protocolos | ✅ | Lista curada manualmente |
| Filtro de token único | ✅ | `len(tokens) == 1` |
| TVL mínimo | ✅ | $500,000 |
| APY máximo (anti-outlier) | ✅ | 500% |
| Flag de cambio brusco de APY | ✅ | ±50% en 24h |
| Detección de lockup | ✅ | Detecta Convex, Votium, veCRV, Velo |
| Origen del rendimiento | ✅ | Muestra fees reales / mixto / solo rewards |
| Alerta de cambio de TVL | ✅ | ±20% en 7d |
| Tendencia APY vs media 30d | ✅ | Indicadores `📈 📉 ➡️` |
| Ejecución de transacciones | ❌ | No implementado — solo lectura |

---

## `/analizar` — Análisis de riesgo con IA

Usa Claude API con búsqueda web en tiempo real (`web_search_20250305`) para analizar el riesgo de un protocolo.

**Flujo:**
1. Claude recibe los datos del pool (protocolo, token, red, APY, TVL)
2. Busca en web: `{protocolo} DeFi audit security` y `{protocolo} DeFi hack exploit`
3. Construye el veredicto con resultados reales
4. La respuesta incluye un campo `FUENTES:` con los links consultados

**Ejemplo de respuesta:**
```
✅ Riesgo: Bajo

PROTOCOLO: aave-v3 / USDC / Base
APY: 5.10% | TVL: $73M

ANÁLISIS:
Aave v3 es uno de los protocolos más auditados en DeFi...

RIESGO PRINCIPAL:
Riesgo de smart contract inherente a cualquier protocolo on-chain.

FUENTES:
- https://...
```

---

## Ejemplos de mensajes en Telegram

```
1. USDC — avantis
   APY: 9.67%
   TVL: $89,727,924 (Grande)
   Fuente: Fees reales (orgánico)
   Riesgo: Bajo
   Lockup: No — salida libre
```

```
2. USDC — aave-v3  APY +62.3% en 24h ⚠️
   APY: 5.10%
   TVL: $73,200,403 (Grande)
   Fuente: Fees reales (orgánico)
   Riesgo: Bajo
```

---

## Archivos del proyecto

| Archivo | Descripción |
|---|---|
| `bot.py` | Lógica principal — filtros, clasificación, handlers de Telegram |
| `analizador.py` | Módulo de análisis de riesgo con Claude API + búsqueda web |
| `filtros.py` | Panel interactivo de filtros inline |
| `diagnostico.py` | Script de debug — corre sin Telegram y muestra qué pools pasan/fallan cada filtro |
| `requirements.txt` | Dependencias del proyecto |
| `.env.example` | Template de variables de entorno |
| `defi-stable-bot.service` | Definición del servicio systemd |

---

## Troubleshooting

| # | Síntoma | Causa probable | Solución |
|---|---|---|---|
| 1 | `ModuleNotFoundError` | venv no activado o deps no instaladas | `./venv/bin/pip install -r requirements.txt` |
| 2 | Bot no responde en Telegram | Token incorrecto | Verificar `.env` y que el service esté activo |
| 3 | `/oportunidades` no muestra nada | Filtros o error en DeFiLlama | Correr `diagnostico.py` y ver resumen de filtros |
| 4 | systemd status = failed | Error al iniciar | `journalctl -u defi-stable-bot -n 20` |
| 5 | Bot iniciado pero no responde | Token de otro bot en `.env` | Verificar que el token corresponde a este bot |
| 6 | Aparecen pools de liquidez | Código viejo en el servidor | Reemplazar `bot.py` y reiniciar el service |
| 7 | `/auto` no llega a horario | Servidor reiniciado y no se reactivó | Mandar `/auto` de nuevo en Telegram |
| 8 | Error de conexión a DeFiLlama | Red o timeout | Cache de 120s lo absorbe si es transitorio |
| 9 | Token no cargado en systemd | Falta `EnvironmentFile=` en el `.service` | Verificar el `.service` |
| 10 | `Falta TELEGRAM_BOT_TOKEN` al arrancar | `.env` vacío o mal path | Verificar ruta en `EnvironmentFile=` del `.service` |

---

## Seguridad

- `.env` fuera de git y fuera de backup (`chmod 600 .env`)
- `.gitignore` incluye `venv/`, `.env`, `__pycache__/`
- El bot es solo lectura — no firma ni ejecuta transacciones
- DeFiLlama no requiere API key — sin credenciales que rotar

---

## Próximas mejoras

| Prioridad | Mejora |
|---|---|
| Media | Persistencia de `/filtros` entre reinicios (actualmente se resetea con el bot) |
| Media | Confirmar flag de TVL en condición real ±20% |
| Baja | Soporte para más redes (Optimism, Polygon) |

---

## Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-02-28 | Versión inicial en producción como service systemd |
| 2026-02-28 | Filtro estricto: solo depósitos de un único activo, whitelist de protocolos |
| 2026-02-28 | Reporte separado por red, top 10 por mensaje, reporte automático 8am/8pm |
| 2026-02-28 | Excluidos: Merkl, Beefy, pools de liquidez |
| 2026-03-01 | `/analizar` con Claude API para análisis de riesgo por protocolo |
| 2026-03-01 | `/ayuda` con menú completo y guía de lectura del reporte |
| 2026-03-01 | Tendencia APY vs media 30d en cada pool |
| 2026-03-01 | Flag TVL: alerta si cambia ±20% en 7d |
| 2026-03-01 | `/filtros` con panel interactivo inline |
| 2026-03-01 | Búsqueda web en tiempo real en `/analizar` |

---

## Licencia

MIT
