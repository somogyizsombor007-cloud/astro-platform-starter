htmlhttps://github.com/somogyizsombor007-cloud/astro-platform-starter/tree/main{
  "name": "therealvest-shop",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "cors": "^2.8.5",
    "dotenv": "^16.0.3",
    "stripe": "^12.14.0",
    "body-parser": "^1.20.2"
  },
  "devDependencies": {
    "nodemon": "^2.0.22"
  }
}# Astro on Netlify Platform Starter

[Live Demo](https://astro-platform-starter.netlify.app/)

A modern starter based on Astro.js, Tailwind, and [Netlify Core Primitives](https://docs.netlify.com/core/overview/#develop) (Edge Functions, Image CDN, Blob Store).

## Astro Commands

All commands are run from the root of the project, from a terminal:

| Command                   | Action                                           |
| :------------------------ | :----------------------------------------------- |
| `npm install`             | Installs dependencies                            |
| `npm run dev`             | Starts local dev server at `localhost:4321`      |
| `npm run build`           | Build your production site to `./dist/`          |
| `npm run preview`         | Preview your build locally, before deploying     |
| `npm run astro ...`       | Run CLI commands like `astro add`, `astro check` |
| `npm run astro -- --help` | Get help using the Astro CLI                     |

## Deploying to Netlify

[![Deploy to Netlify](https://www.netlify.com/img/deploy/button.svg)](https://app.netlify.com/start/deploy?repository=https://github.com/netlify-templates/astro-platform-starter)

## Developing Locally

| Prerequisites                                                                |
| :--------------------------------------------------------------------------- |
| [Node.js](https://nodejs.org/) v18.14+.                                      |
| (optional) [nvm](https://github.com/nvm-sh/nvm) for Node version management. |

1. Clone this repository, then run `npm install` in its root directory.

2. For the starter to have full functionality locally (e.g. edge functions, blob store), please ensure you have an up-to-date version of Netlify CLI. Run:

```
npm install netlify-cli@latest -g
```

3. Link your local repository to the deployed Netlify site. This will ensure you're using the same runtime version for both local development and your deployed site.

```
netlify link
```

4. Then, run the Astro.js development server via Netlify CLI:

```
netlify dev
```

If your browser doesn't navigate to the site automatically, visit [localhost:8888](http://localhost:8888).
http://localhost:3000require('dotenv').config();
const express = require('express');
const Stripe = require('stripe');
const cors = require('cors');
const bodyParser = require('body-parser');

const app = express();
const stripe = Stripe(process.env.STRIPE_SECRET_KEY);
const PORT = process.env.PORT || 4242;
const FRONTEND_URL = process.env.FRONTEND_URL || 'http://localhost:3000';

app.use(cors());
app.use(express.json());

let products = [];

app.post('/api/add-product', (req, res) => {
  const { title, price, img, sheinLink } = req.body;
  if (!title || !price) return res.status(400).json({ error: 'Missing title or price' });
  const id = 'p' + Date.now();
  const p = { id, title, price: Math.round(Number(price)), img: img || '', sheinLink: sheinLink || '' };
  products.push(p);
  res.json({ success: true, product: p });
});

app.get('/api/products', (req, res) => {
  res.json(products);
});

app.post('/api/create-checkout-session', async (req, res) => {
  const { productId } = req.body;
  const product = products.find(p => p.id === productId);
  if (!product) return res.status(404).json({ error: 'Product not found' });

  try {
    const session = await stripe.checkout.sessions.create({
      payment_method_types: ['card'],
      line_items: [{
        price_data: {
          currency: 'huf',
          product_data: { name: product.title },
          unit_amount: product.price
        },
        quantity: 1
      }],
      mode: 'payment',
      success_url: `${FRONTEND_URL}/success?session_id={CHECKOUT_SESSION_ID}`,
      cancel_url: `${FRONTEND_URL}/cancel`,
    });
    res.json({ url: session.url });
  } catch (err) {
    console.error('Stripe error', err);
    res.status(500).json({ error: 'Stripe error' });
  }
});

app.post('/webhook', bodyParser.raw({ type: 'application/json' }), (req, res) => {
  const sig = req.headers['stripe-signature'];
  let event;

  try {
    event = stripe.webhooks.constructEvent(req.body, sig, process.env.STRIPE_WEBHOOK_SECRET);
  } catch (err) {
    console.error('Webhook signature verification failed.', err.message);
    return res.status(400).send(`Webhook Error: ${err.message}`);
  }

  if (event.type === 'checkout.session.completed') {
    const session = event.data.object;
    console.log('Payment succeeded for session:', session.id, 'amount_total:', session.amount_total);
  }

  res.json({ received: true });
});

app.listen(PORT, () => {
  console.log(`Server listening on port ${PORT}`);
});<!doctype html>
<html>
<head>
<meta charset="utf-8"/>
<title>THEREALVEST - Shop</title>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<style>
body{font-family:Arial,sans-serif;background:#0b0b0b;color:#fff;padding:18px}
.wrap{max-width:900px;margin:auto}
.header{display:flex;align-items:center;justify-content:space-between}
.brand{color:#ff2d2d;font-weight:800;font-size:24px}
.admin{background:#111;padding:12px;border-radius:10px;margin:16px 0}
input,button{padding:8px;margin:6px 0;border-radius:8px;border:1px solid #333}
.grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(220px,1fr));gap:12px}
.card{background:#111;padding:12px;border-radius:10px}
a { color: #fff; text-decoration:none }
</style>
</head>
<body>
<div class="wrap">
  <div class="header">
    <div class="brand">THEREALVEST</div>
    <div style="color:#bbb">Fekete • Piros • Fehér</div>
  </div>

  <div class="admin" id="adminPanel">
    <h3>Admin: új termék</h3>
    <input id="title" placeholder="Termék neve"/><br/>
    <input id="price" placeholder="Ár HUF (pl. 14990)" type="number"/><br/>
    <input id="img" placeholder="Kép URL (opcionális)"/><br/>
    <input id="link" placeholder="Shein link (opcionális)"/><br/>
    <button id="addBtn">Hozzáad</button>
  </div>

  <div id="products" class="grid"></div>
</div>

<script>
const API_BASE = 'http://localhost:4242';

async function fetchProducts(){
  try{
    const res = await fetch(API_BASE + '/api/products');
    const list = await res.json();
    render(list);
  }catch(e){ console.error(e) }
}

function render(list){
  const container = document.getElementById('products');
  container.innerHTML = '';
  list.forEach(p=>{
    const div = document.createElement('div');
    div.className = 'card';
    div.innerHTML = `
      ${p.img ? `<img src="${p.img}" style="width:100%;height:140px;object-fit:cover;border-radius:8px;margin-bottom:8px">` : ''}
      <h4 style="margin:6px 0">${p.title}</h4>
      <div style="color:#ff6b6b;font-weight:700">${p.price} Ft</div>
      <div style="margin-top:8px;display:flex;gap:6px">
        <button onclick="buy('${p.id}')">Megveszem</button>
        <a href="${p.sheinLink||'#'}" target="_blank" style="margin-left:auto;color:#9aa">Eredeti</a>
      </div>
    `;
    container.appendChild(div);
  });
}

document.getElementById('addBtn').addEventListener('click', async ()=>{
  const title = document.getElementById('title').value.trim();
  const price = document.getElementById('price').value.trim();
  const img = document.getElementById('img').value.trim();
  const sheinLink = document.getElementById('link').value.trim();
  if(!title || !price){ alert('Add meg a címet és az árat'); return; }
  await fetch(API_BASE + '/api/add-product', {
    method:'POST', headers:{'Content-Type':'application/json'},
    body: JSON.stringify({ title, price, img, sheinLink })
  });
  document.getElementById('title').value=''; document.getElementById('price').value=''; document.getElementById('img').value=''; document.getElementById('link').value='';
  fetchProducts();
});

async function buy(productId){
  try{
    const res = await fetch(API_BASE + '/api/create-checkout-session', {
      method:'POST', headers:{'Content-Type':'application/json'},
      body: JSON.stringify({ productId })
    });
    const data = await res.json();
    if(data.url){
      window.location.href = data.url;
    }else{
      alert('Hiba a fizetés indításakor');
      console.error(data);
    }
  }catch(e){ console.error(e); alert('Hiba') }
}

fetchProducts();
</script>
</body>
</html>