# यशोदा गृहचव — Website

Static storefront (HTML + CSS + vanilla JS). No build step. Orders are sent to WhatsApp **+91 90498 03588**.

## Files

| File | Purpose |
|---|---|
| `index.html` | Page layout and all text |
| `styles.css` | Design (colours from the logo: maroon, leaf green, haldi gold) |
| `app.js` | Menu, cart, tiered discount, WhatsApp order message |
| `products.json` | **Edit this for prices, products, best-sellers** |
| `offers.json` | Discount on/off and levels. Changed from the hidden admin panel |
| `admin.json` | Created by admin setup. Holds the GitHub token, encrypted with the admin password |
| `assets/` | Logo, FSSAI logo, favicon, share image, product photos |
| `.nojekyll` | Tells GitHub Pages to serve files as-is |

## Common edits (only `products.json`)

- **Change a price:** `"rates" → "पापड" → "तांदूळ": 320`. The value is ₹ per kg.
- **Add a missing price:** add the item under its group in `rates`, e.g. `"दादर": 330`.
- **Add a product:** add its name to the category's `"i"` list, then add its rate.
- **Puran poli flyer price:** `"special" → "amount": 100`. Delete `amount` to show "दर विचारा".
- **Photo for an item:** `"itemImages" → "दिवाळी फराळ|चकली": "chakli"`. This points to `assets/img/chakli.webp`. Items without an entry use their category photo.
- **Best-sellers row:** `"bestsellers"` uses `"Category|Item"` keys.

## Discount / offers (admin)

The discount is applied to the **final order amount**, based on the **total kg in the order**. The default is 5 kg or more → 5%, and 10 kg or more → 10%.

- The total kg counts every product sold by the kilo.
- The % is applied to the whole priced amount, including नग items such as नागली पापड.
- Parcel charges are not discounted.

When the offer is turned **off**, every discount mention disappears from the site: the banner, ticker, menu link, FAQ, cart lines and WhatsApp message.

### One-time setup (after the site is live on GitHub Pages)

1. **Create a GitHub token.**
   1. Go to GitHub → profile picture → **Settings** → **Developer settings** → **Personal access tokens** → **Fine-grained tokens** → **Generate new token**.
   2. Name it `yashoda-offers`. For expiry, choose the longest period offered and write down the date.
   3. Under **Repository access**, choose **Only select repositories** → `yashoda-gruhchav`.
   4. Under **Permissions → Repository permissions → Contents**, choose **Read and write**.
   5. Generate the token and copy it. It starts with `github_pat_`.
2. **Open the admin panel.** Open the live website and scroll to the very bottom. **Double-tap (phone) or double-click (computer) the bottom-right corner of the footer**, just to the right of the "© … सर्व हक्क राखीव" line. Nothing is visible there.
3. **Run setup.** Tap **"पहिल्यांदा सेटअप / पासवर्ड बदला"** and fill in:
   - GitHub username
   - repository: `yashoda-gruhchav`
   - branch: `main`
   - the token
   - a new admin password (at least 8 characters)

   Then tap **सेटअप सेव्ह करा**.

The token is stored only in encrypted form, in `admin.json`, and the password is never stored anywhere.

### Daily use

1. Double-tap the bottom-right corner of the footer and enter the password.
2. Use the **ऑफर / सूट** switch to turn the offer on or off.
3. Edit the levels: minimum total kg and the %. Use **+ नवीन स्तर जोडा** to add a level and **×** to remove one.
4. Tap **सेव्ह करा व वेबसाइटवर लागू करा**. All visitors see the change within about 1–2 minutes.
5. While you are logged in, a small **⚙ ऑफर** button appears at the bottom-left so you can reopen the panel. Use **लॉगआउट** when you are done.

### Security notes

- Use a strong password, at least 12 characters with numbers and symbols. `admin.json` is public in the repo, so the password is the only thing protecting the token.
- The token can only edit files in this one repository.
- **Forgot the password, or the token expired:** create a new token and run setup again ("पासवर्ड बदला").
- **Lost access completely:** you can always edit `offers.json` directly on GitHub. Set `"enabled": true` or `false` and change the `tiers`.

## Publish on GitHub Pages (free)

1. Create a **public** repository on GitHub, e.g. `yashoda-gruhchav`.
2. Upload **everything in this folder**, keeping the `assets` folder structure. Put `index.html` at the repo root.
3. Go to **Settings → Pages**. Under **Source**, choose **Deploy from a branch**. Set **Branch** to `main`, folder **/ (root)**, then **Save**.
4. After 1–2 minutes the site is live at `https://<your-github-username>.github.io/yashoda-gruhchav/`.

## Custom domain (recommended)

Suggested names. Check availability first on GoDaddy, Hostinger, BigRock or Namecheap.

1. `yashodagruhchav.in` (first choice: short, Indian `.in`)
2. `yashodagruhchav.com`
3. `yashoda-gruhchav.in`
4. `gruhchav.in`

Connect the domain:

1. Buy the domain.
2. In GitHub, go to **Settings → Pages → Custom domain**, enter `yashodagruhchav.in` and **Save**. GitHub creates a `CNAME` file for you.
3. At the domain registrar, open DNS settings and add:
   - Four **A** records for `@`: `185.199.108.153`, `185.199.109.153`, `185.199.110.153`, `185.199.111.153`
   - One **CNAME** record for `www` pointing to `<your-github-username>.github.io`
4. Wait for DNS (from a few minutes up to 24 hours). Then tick **Enforce HTTPS** in GitHub Pages.

## After going live

- Register free on **Google Business Profile** with the same name, address and phone, and add the website link.
- Share the site link in the WhatsApp Business catalogue and status.
- Replace the photos in `assets/img/` with your own product photos. Keep the same file names, in `.webp` format, at 800×600 px. Any free online converter can save a photo as `.webp`.
