# Diego DeFi — Bots & Herramientas

Documentación de los bots y herramientas que desarrollo y corro en mi servidor personal.
Cada carpeta contiene el README completo de un proyecto: qué hace, cómo instalarlo, cómo configurarlo y cómo operarlo.

El código no está publicado — estos READMEs están pensados para que cualquiera pueda armar su propia versión.

---

## Proyectos

| Proyecto | Descripción |
|---|---|
| [defi-stable-bot](./defi-stable-bot/README.md) | Bot de Telegram que reporta oportunidades de depósito en stablecoins en Base y Arbitrum via DeFiLlama. Incluye análisis de riesgo con IA. |
| [plantilla-bot](./plantilla-bot/README.md) | Template para documentar bots de home server |
| [pago-con-cripto](./pago-con-cripto/README.md) | Herramienta estática para cobrar en cripto en un negocio — ARS a USDC/USDT con QR de FluidKey |
| [trading-bot](./trading-bot/README.md) | Bot de trading automático para Binance Spot — estructura genérica adaptable a cualquier par y estrategia |
| [bot-gastos](./bot-gastos/README.md) | Bot de Telegram para registrar gastos diarios con categorías, cuotas automáticas y reportes desde Google Sheets |
| [dashboard-gastos](./dashboard-gastos/README.md) | Panel web para visualizar gastos de bot-gastos — lee directo de Google Sheets, sin base de datos |
---

## Stack general

- Python 3.10+
- Telegram Bot API
- Systemd para servicios permanentes
- DeFiLlama API (pública)
- Claude API (Anthropic) para análisis con IA

---

## Contacto

Si tenés preguntas o sugerencias, abrí un Issue en este repo.
