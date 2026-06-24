# Turning on real Stripe billing

Both apps (`vision.html`, `changelog.html`) ship with a real Stripe checkout path that works on plain GitHub Pages — **no backend, no secret keys, no server to run.** It uses **Stripe Payment Links**. Until you paste your links in, the "Unlock Pro" button stays in **demo mode** (instant unlock, no charge).

## What you get out of the box
- Click "Unlock Pro" → redirect to Stripe's hosted, PCI-compliant checkout (real subscription, real money).
- After paying, Stripe sends the customer back to the app and Pro unlocks.
- Nothing secret is ever committed to the repo.

## Setup (about 10 minutes)

1. **Create a Stripe account** at https://dashboard.stripe.com (your task — it needs your identity/banking).
2. **Create two prices** (Products → Add product):
   - Vision Pro — **$4.99 / month** (recurring) and optionally **$39 / year**.
   - Changelog Pro — **$3.99 / month** and optionally **$24 / year**.
3. **Create a Payment Link** for each price (Payment Links → New). On the **"After payment"** step, choose **"Don't show confirmation page → Redirect to your website"** and set the URL to:
   - Vision monthly/yearly → `https://danielduongg.github.io/changelog/vision.html?upgraded=1`
   - Changelog monthly/yearly → `https://danielduongg.github.io/changelog/changelog.html?upgraded=1`
4. **Paste the links** into the `STRIPE` config near the top of each file's `<script>`:
   ```js
   // vision.html
   const STRIPE = {
     MONTHLY: "https://buy.stripe.com/your-vision-monthly",
     YEARLY:  "https://buy.stripe.com/your-vision-yearly"
   };
   ```
   ```js
   // changelog.html
   const STRIPE = {
     MONTHLY: "https://buy.stripe.com/your-changelog-monthly",
     YEARLY:  "https://buy.stripe.com/your-changelog-yearly"
   };
   ```
5. Re-upload the two files. Done — the button now reads "Subscribe — $4.99/mo" and charges for real.

Payment Link URLs and Stripe **publishable** keys are public and safe to commit. Your Stripe **secret** key is never used here and must never be put in this repo.

## Honest limitation (read this)

On a static host the app trusts `?upgraded=1` to flip Pro and stores it in the browser. That's perfect for launch and low-stakes "Pro," but it is **not tamper-proof** — a determined user could set the flag manually, and entitlement doesn't follow them to a new device. To make Pro genuinely enforced you need a tiny backend (one webhook + one verify call). GitHub Pages can't run that; move hosting to **Netlify, Vercel, or Cloudflare** (the static files work there unchanged) and add the function below.

## Appendix — tamper-proof entitlement (serverless, optional)

Host the same files on Netlify/Vercel, keep your **secret** key in an env var (never in the repo), and add two pieces:

**1. Webhook** — record who paid. Stripe → Developers → Webhooks → send `checkout.session.completed` and `customer.subscription.*` to `/api/stripe-webhook`. Store `{ email/customerId → active }` in a small KV/DB.

**2. Verify on load** — the app asks your function "is this customer active?" instead of trusting the URL flag.

```js
// /api/verify.js  (Vercel-style; Netlify is the same idea)
import Stripe from "stripe";
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY); // env var only

export default async function handler(req, res) {
  const { session_id } = req.query;            // returned by Checkout success URL
  if (!session_id) return res.status(400).json({ active: false });
  const s = await stripe.checkout.sessions.retrieve(session_id, { expand: ["subscription"] });
  const active = s.payment_status === "paid" &&
                 s.subscription && s.subscription.status === "active";
  res.json({ active, customer: s.customer });   // app unlocks only if active === true
}
```

Then set the Payment Link success URL to `...vision.html?session_id={CHECKOUT_SESSION_ID}`, and on load call `/api/verify?session_id=…` before unlocking. With that in place, "Pro" actually means paid.

---
*Test mode tip: use Stripe **test mode** + card `4242 4242 4242 4242` to try the whole flow without real charges before going live.*
