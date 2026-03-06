# 💳 Pago con Cripto — ARS → USDC/USDT

Herramienta estática para cobrar en cripto en un negocio físico o digital.
Convierte un precio en pesos argentinos a USDC/USDT usando cotización en tiempo real,
y muestra el QR de pago de [FluidKey](https://fluidkey.com) del comerciante.

> **Bien público.** Tomá el código, adaptalo a tu negocio, usalo como quieras.
> Solo se pide que menciones al creador: [@diegodefieth-hash](https://github.com/diegodefieth-hash)

---

## ¿Qué hace?

1. El comerciante ingresa el monto en ARS.
2. La página obtiene la cotización USDC/ARS como **promedio en tiempo real** de Buenbit, Ripio, Belo y Fiwind vía [CriptoYA](https://criptoya.com).
3. Calcula el equivalente en USDC/USDT y agrega un buffer del 0.5% para cubrir variaciones de precio.
4. Muestra el monto con un botón de copiar y el **QR nativo de FluidKey** del comerciante para que el cliente pague desde su wallet.

No tiene backend. Todo corre en el navegador. No procesa ni toca fondos.

---

## Archivos

| Archivo | Descripción |
|---|---|
| `checkout.html` | Página principal del cobro |
| `faq.html` | FAQ: cómo funciona, qué es FluidKey, redes, wallets |
| `README.md` | Este archivo |

---

## Cómo usar

### Opción A — Abrir directo en el navegador

No requiere servidor. Descargá los dos archivos HTML y abrí `checkout.html` en el navegador.

### Opción B — Servir desde el servidor (recomendado para uso en negocio)

```bash
# Clonar en el servidor
git clone https://github.com/diegodefieth-hash/diegodefieth-docs.git
cd diegodefieth-docs/pago-con-cripto

# Servir con cualquier servidor estático, por ejemplo:
python3 -m http.server 8080
# → Abrir http://localhost:8080/checkout.html
```

O subir los archivos a GitHub Pages, Netlify, Vercel, o cualquier hosting estático.

---

## Cómo adaptar para tu negocio

Todo lo configurable está agrupado al inicio del `<script>` en `checkout.html`:

```javascript
const BUFFER_PCT = 0.5; // buffer sobre el monto (%) para cubrir variaciones de precio

const SOURCES = [
  { id: 'buenbit', label: 'Buenbit', url: 'https://criptoya.com/api/buenbit/USDC/ARS/0.1', field: 'totalBid' },
  { id: 'ripio',   label: 'Ripio',   url: 'https://criptoya.com/api/ripio/USDC/ARS/0.1',   field: 'totalBid' },
  { id: 'belo',    label: 'Belo',    url: 'https://criptoya.com/api/belo/USDC/ARS/0.1',    field: 'totalBid' },
  { id: 'fiwind',  label: 'Fiwind',  url: 'https://criptoya.com/api/fiwind/USDC/ARS/0.1',  field: 'totalBid' },
];
```

Para cambiar tu URL de FluidKey: ingresala en el campo "URL de FluidKey" dentro de la página y presioná "Guardar". Se guarda en el navegador automáticamente para futuros cobros.

---

## Cotización

La tasa ARS/USDC se obtiene en tiempo real de [CriptoYA](https://criptoya.com) consultando el precio **bid** (precio de venta) de cada exchange. Si alguno falla, se excluye del promedio automáticamente. Si todos fallan (por ejemplo por CORS en ciertos navegadores), el campo de tasa queda editable para ingresarla manualmente.

---

## Tecnologías

- HTML + CSS + JavaScript vanilla — sin frameworks, sin dependencias
- [CriptoYA API](https://criptoya.com/api) — cotización ARS/USDC en tiempo real
- [FluidKey](https://fluidkey.com) — QR de pago con stealth addresses
- Tailwind CDN — estilos (solo en el prototipo HTML original, la versión final usa CSS propio)

---

## Redes soportadas para el pago

Base · Arbitrum One · Optimism · Polygon · Gnosis Chain

Tokens aceptados: **USDC** y **USDT**

---

## Troubleshooting

| Síntoma | Causa probable | Solución |
|---|---|---|
| La cotización no carga | CORS bloqueado por el navegador | Ingresá la tasa manualmente en el campo habilitado |
| El QR no se ve | El navegador bloqueó el iframe de FluidKey | La página ofrece un link directo para abrir FluidKey en nueva pestaña |
| El chip de un exchange aparece en rojo | Ese exchange no respondió la API | Se excluye del promedio automáticamente, no afecta el funcionamiento |
| El cliente pagó pero no llegó | Pagó desde un exchange (retiro no instantáneo) | Pedirle al cliente que pague siempre desde su wallet self-custodial |

---

## Licencia y créditos

Herramienta de uso libre como bien público.
Podés usarla, modificarla y redistribuirla para tu negocio sin restricciones.

**Única condición:** si la usás o la mostrás, mencioná al creador original:

> Creado por [@diegodefieth-hash](https://github.com/diegodefieth-hash)

---

## Historial de cambios

| Fecha | Cambio |
|---|---|
| 2025-03-06 | Versión inicial pública — checkout + FAQ |
| 2025-03-06 | Cotización como promedio de Buenbit, Ripio, Belo y Fiwind |
| 2025-03-06 | Fallback manual si CORS falla, monto copiable, botón Nuevo cobro |
