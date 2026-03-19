# Fluid Lite — Diagrama interactivo USD Vault

Diagrama visual e interactivo del sistema completo de **Fluid Lite fLiteUSD**, el vault de stablecoins de Instadapp.

## 🌐 Ver diagrama

👉 **[diegodefieth-hash.github.io/diegodefieth-docs/fluid-lite-diagram](https://diegodefieth-hash.github.io/diegodefieth-docs/fluid-lite-diagram/)**
## ¿Qué muestra?

El diagrama cubre el flujo completo desde que depositás USDC hasta que retirás con yield:

- **Vault principal** — fLiteUSD (ERC-4626), Ethereum Mainnet, base rate 6%
- **Strategy Handler** — distribuye capital entre cadenas via CCIP y LayerZero
- **Cadenas** — Ethereum, Arbitrum y Plasma
- **Activos yield-bearing** utilizados:
  - `sUSDe` — Ethena · funding rates de perpetuos + ETH staking · 7–18% variable
  - `syrupUSDC` / `syrupUSDT` — Maple Finance · préstamos institucionales sobrecolateralizados · ~7% estable
  - `sUSDai` — USD.AI (Permian Labs) · préstamos a operadores de GPU / IA · RWA · 10–15% APR
- **Loop de leverage recursivo** — colateral → préstamo → swap → más colateral
- **Reconciliación semanal** — cada domingo, ~3–4 horas, actualiza el exchange rate
- **Costos y riesgos** — exit fee 0.05%, sin performance fee, riesgos por activo
## Nota sobre sUSDai

sUSDai requiere ~30 días para unstakear directamente en USD.AI. Fluid resuelve esto sin que el usuario espere nada: usa sUSDai como colateral en loops de lending y, cuando necesita liquidez, lo swapea en su propio DEX, donde Fluid es el principal hub de liquidez para este activo.

## Fuentes

- [lite.guides.instadapp.io](https://lite.guides.instadapp.io)
- [usd.ai](https://usd.ai)
- [maple.finance](https://maple.finance)
- [ethena.fi](https://ethena.fi)
