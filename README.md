# laskatel - Portfolio

Fullstack developer. Working with Telegram APIs, AI integrations, automation, API's and web backends.

# About Me

**API integration specialist**

I excel at quickly navigating technical documentation and implementing third-party services. Ready to connect any API - from payment gateways to AI models - with clean, maintainable code


**Automation & Backend Deployment Specialist**

I specialize in end-to-end process automation, deploying production-ready backends on any stack, and building client-facing automation systems. From internal workflows to scalable cloud services and automated client communications, I deliver reliable, well-documented solutions that run smoothly out of the box


**Machine Learning**

I specialize in developing, fine-tuning, and deploying AI models - from LLMs and vision systems to custom ML pipelines. I turn research-grade models into production-ready solutions.



P.S
You can request additional screenshots or code from me on Telegram, which is listed at the end of the portfolio.
---

## Projects

# 1. Studio Myata - AI Photo Generation Platform

### Studio Myata — AI Interior & Exterior Redesign Platform

Full-stack web service for a real photo studio. Users upload a photo of a room, facade or garden, pick a style — and get back an AI-generated redesign in under a minute. Everything is production-deployed: [ai.studio-mint.ru](https://ai.studio-mint.ru)


## What's inside

- AI generation via **Replicate** (`flux-kontext-max`) — interiors, facades, landscapes
- **OAuth 2.0** login — Yandex and VK, popup-based, session-persisted
- **Credit system** — users buy credits via UnitPay, credits are deducted per generation
- **Gallery** — users choose to publish, others can like; shared links via unique tokens
- **Promo codes** — admin creates codes with credit value and usage cap
- **Admin panel** — full CRUD for users, generations, news, styles, pricing, site images, settings
- **Maintenance mode** — toggleable from admin panel, IP whitelist for bypass
- **Cloudinary** for storage, with automatic local filesystem fallback
- **CBR exchange rate** — fetched from Central Bank API, cached 6h, used for USD pricing display


## Stack

| Layer | Tech |
|---|---|
| Runtime | Node.js 20 (ESM) |
| Framework | Express |
| Database | better-sqlite3 |
| Auth | Passport.js (passport-yandex, passport-vkontakte) |
| Sessions | express-session + connect-sqlite3 |
| AI | Replicate API — `flux-kontext-max` |
| Payments | UnitPay |
| Storage | Cloudinary / local (auto-detected from env) |
| Frontend | Vanilla JS + HTML/CSS (no bundler, intentional) |
| Deploy | nginx reverse proxy over Unix socket |


## OAuth — Yandex & VK

Strategies are registered dynamically at startup — if the env vars for a provider are missing, the routes return `503` instead of crashing. Both work the same way internally.

```js
// oauth.js — Yandex strategy setup
if (process.env.YANDEX_CLIENT_ID && process.env.YANDEX_CLIENT_SECRET) {
  passport.use('yandex', new YandexStrategy({
    clientID:     process.env.YANDEX_CLIENT_ID,
    clientSecret: process.env.YANDEX_CLIENT_SECRET,
    callbackURL:  BASE_URL + '/api/auth/yandex/callback',
  }, (accessToken, refreshToken, profile, done) => {
    done(null, {
      imya:   profile.displayName || profile.username,
      email:  profile.emails?.[0]?.value || null,
      avatar: profile.photos?.[0]?.value || null,
    });
  }));
}
```

The login opens in a popup. After the provider redirects back, the callback upserts the user into the DB, saves the session, then does a `postMessage` to close the popup and update the parent page — no full reload:

```js
// oauth.js — after successful auth
function posleOAuth(req, res, profil) {
  const polz = naytiIliSozdat(profil);
  req.session.polzovatel = { id: polz.id };
  req.session.save(() => {
    res.send(`<!DOCTYPE html><html><body><script>
      if (window.opener) {
        window.opener.postMessage({
          tip: 'oauth_ok',
          polzovatel: ${JSON.stringify(polzPubl(polz))}
        }, window.location.origin);
        window.close();
      } else {
        window.location.href = '/';
      }
    </script></body></html>`);
  });
}
```

New users are created automatically with a signup credit bonus pulled from the settings table. If a returning user logs in without an avatar, it gets filled in:

```js
// oauth.js — upsert user on login
function naytiIliSozdat({ imya, email, avatar }) {
  let polz = email
    ? baza.prepare('SELECT * FROM polzovateli WHERE email = ?').get(email)
    : null;

  if (!polz) {
    const kredity = parseInt(getNastroyka('kredity_za_registraciyu') || '10');
    const fakeEmail = email || ('oauth_' + Date.now() + '@oauth.local');
    const r = baza.prepare(
      'INSERT INTO polzovateli (email, imya, avatar, kredity) VALUES (?, ?, ?, ?)'
    ).run(fakeEmail, imya || 'Пользователь', avatar || null, kredity);
    polz = baza.prepare('SELECT * FROM polzovateli WHERE id = ?').get(r.lastInsertRowid);
  } else if (avatar && !polz.avatar) {
    baza.prepare('UPDATE polzovateli SET avatar = ? WHERE id = ?').run(avatar, polz.id);
    polz = baza.prepare('SELECT * FROM polzovateli WHERE id = ?').get(polz.id);
  }
  return polz;
}
```

## AI Generation — Replicate / Flux Kontext Max

Each call sends the user's photo URL + a composed prompt to Replicate, then polls until the prediction finishes. The model preserves the room's geometry — it's not replacing the photo, it's restyling it.

```js
// replicate.js — API call
async function sozdatPro(fotoUrl, polniyPrompt) {
  const res = await fetch(
    'https://api.replicate.com/v1/models/black-forest-labs/flux-kontext-max/predictions',
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${TOKEN}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        input: {
          prompt:            polniyPrompt,
          input_image:       fotoUrl,
          aspect_ratio:      'match_input_image',
          output_format:     'jpg',
          safety_tolerance:  2,
          prompt_upsampling: true,
          seed: Math.floor(Math.random() * 2147483647),
        },
      }),
    }
  );
  // polls /predictions/{id} until status === 'succeeded' or 'failed'
}
```

Prompts are composed from a base template + style fragment + room type fragment. Room types inject detailed furniture lists — generic prompts leave the space empty, which was a real problem before adding these:

```js
// replicate.js — room type prompt fragments (excerpt)
const POMESHCHENIE_PROMPTY = {
  'Спальня': 'bedroom fully furnished with double or king bed with headboard and bedding, ' +
             'two bedside tables with lamps, large wardrobe, dresser, mirror, curtains and decorative items on surfaces, ',
  'Кухня':   'kitchen fully equipped with cabinetry, countertops, refrigerator, oven, ' +
             'kitchen island or dining table with chairs, pendant lights, backsplash tiles, small appliances, ',
  // ...
};

// Style fragments (12+ per category)
const STIL_PROMPTY = {
  'japandi':  'japandi interior, Japanese minimalism meets Scandinavian warmth, natural wood, ' +
              'wabi-sabi textures, muted earth tones, handcrafted ceramics, bamboo accents',
  'blektek':  'dark luxury interior, black walls, dramatic matte finishes, moody atmospheric lighting, ' +
              'gold and bronze accents, sophisticated noir aesthetic',
  'loft':     'industrial loft interior, exposed brick walls, concrete ceiling, steel beams, ' +
              'Edison bulbs, reclaimed wood floors, urban raw aesthetic',
  // ...
};
```

Generation is fire-and-forget from the user's perspective — response returns immediately with a `generaciya_id`, the client polls `/api/zadacha/:id`. Credits are refunded if Replicate throws:

```js
// api.js — POST /api/generirovat (background part, runs after response)
baza.prepare('UPDATE polzovateli SET kredity = kredity - ? WHERE id = ?').run(stoimost, pol.id);
const gen = baza.prepare(
  'INSERT INTO generacii (polzovatel_id, stil_ids, foto_do, prompt, tip_dizayna, status) VALUES (?, ?, ?, ?, ?, ?)'
).run(pol.id, stilIdsStr, fotoUrl, stilPrompt, tip_dizayna, 'ozhidanie');

res.json({ generaciya_id: gen.lastInsertRowid, kredity: polObnovl.kredity });

(async () => {
  try {
    baza.prepare("UPDATE generacii SET status = 'v_rabote' WHERE id = ?").run(genId);
    const fotoPosl = await sozdat(fotoUrl, stilPrompt, tip_dizayna, tipModel, tip_pomeshcheniya);
    const cloudUrl = await zagruzitPoURL(fotoPosl, 'rd/rezultat');
    baza.prepare("UPDATE generacii SET foto_posle = ?, status = 'gotovo' WHERE id = ?").run(cloudUrl, genId);
  } catch (e) {
    baza.prepare("UPDATE generacii SET status = 'oshibka', oshibka_tekst = ? WHERE id = ?")
      .run(e.message.slice(0, 500), genId);
    baza.prepare('UPDATE polzovateli SET kredity = kredity + ? WHERE id = ?').run(stoimost, pol.id);
  }
})();
```


## Payments — UnitPay

Payment flow: client calls `/api/platezh/sozdat` → server creates a pending payment record, builds a signed redirect URL, returns it → client opens UnitPay → UnitPay calls `/api/platezh/handler` with `check` then `pay`.

**Signature for payment form:**

```js
// unitpay.js
export function formSignature(account, currency, desc, sum) {
  const str = [account, currency, desc, sum, secretKey].join('{up}');
  return crypto.createHash('sha256').update(str).digest('hex');
}
```

**Webhook handler** — verifies IP whitelist, then SHA-256 signature, then processes `check` / `pay` / `error`:

```js
// api.js — GET /api/platezh/handler
router.get('/platezh/handler', (req, res) => {
  const { method, params = {} } = req.query;
  const p = typeof params === 'object' ? params : {};

  // 1. IP check
  const ip = (req.headers['x-forwarded-for'] || req.socket.remoteAddress || '').split(',')[0].trim();
  if (!isUnitpayIp(ip)) return res.json({ error: { message: 'Forbidden IP' } });

  // 2. Signature check
  if (!verifyHandlerSignature(method, p)) return res.json({ error: { message: 'Неверная подпись' } });

  // 3. CHECK — verify the user exists before payment proceeds
  if (method === 'check') {
    const pol = baza.prepare('SELECT id FROM polzovateli WHERE id = ?').get(parseInt(p.account));
    if (!pol) return res.json({ error: { message: 'Пользователь не найден' } });
    return res.json({ result: { message: 'Запрос успешно обработан' } });
  }

  // 4. PAY — credit the account, wrapped in a transaction
  if (method === 'pay') {
    // idempotency: already processed — just return success
    const existing = baza.prepare('SELECT * FROM platezhi WHERE unitpay_id = ?').get(p.unitpayId);
    if (existing?.status === 'oplachen') return res.json({ result: { message: 'Запрос успешно обработан' } });

    const platezh = baza.prepare(
      "SELECT * FROM platezhi WHERE polzovatel_id = ? AND status = 'ozhidanie' AND ABS(summa - ?) < 0.5 ORDER BY id DESC LIMIT 1"
    ).get(parseInt(p.account), parseFloat(p.orderSum));

    if (!platezh) return res.json({ error: { message: 'Платёж не найден' } });

    baza.prepare('BEGIN').run();
    try {
      baza.prepare('UPDATE polzovateli SET kredity = kredity + ? WHERE id = ?')
        .run(platezh.kredity, parseInt(p.account));
      baza.prepare("UPDATE platezhi SET status = 'oplachen', unitpay_id = ? WHERE id = ?")
        .run(p.unitpayId, platezh.id);
      baza.prepare('COMMIT').run();
    } catch (e) {
      baza.prepare('ROLLBACK').run();
      return res.json({ error: { message: 'Ошибка БД: ' + e.message } });
    }

    return res.json({ result: { message: 'Запрос успешно обработан' } });
  }
});
```

Signature verification on the webhook side:

```js
// unitpay.js
export function verifyHandlerSignature(method, params) {
  const sorted = Object.keys(params)
    .filter(k => k !== 'signature')
    .sort()
    .map(k => params[k]);
  const str = [method, ...sorted, secretKey].join('{up}');
  return crypto.createHash('sha256').update(str).digest('hex') === params.signature;
}

export function isUnitpayIp(ip) {
  return ['31.186.100.49', '178.132.203.105', '52.29.152.23', '52.19.56.234', '127.0.0.1']
    .includes(ip);
}
```

Receipts (online-kassa) are passed inline in the payment URL as `cashItems[]` params — no separate integration needed.


## Storage — Cloudinary with Local Fallback

If Cloudinary credentials are in env, uploads go there. If not, files land in `/data/uploads/` and are served by Express. Zero code changes to switch — same function, same return type either way.

```js
// oblako.js
const USE_CLOUDINARY = !!(
  process.env.CLOUDINARY_CLOUD_NAME &&
  process.env.CLOUDINARY_API_KEY    &&
  process.env.CLOUDINARY_API_SECRET
);

export async function zagruzitBuffer(buffer, papka) {
  if (USE_CLOUDINARY) {
    return new Promise((resolve, reject) => {
      const uploadStream = cloudinary.uploader.upload_stream(
        { folder: 'myata/' + papka, resource_type: 'image' },
        (err, result) => {
          if (err) return reject(new Error('Cloudinary: ' + err.message));
          resolve(result.secure_url);  // https://res.cloudinary.com/...
        }
      );
      uploadStream.end(buffer);
    });
  }

  // local fallback — detect format from magic bytes, write to disk
  const ext  = bufferToExt(buffer);
  const name = crypto.randomBytes(16).toString('hex') + ext;
  const rel  = papka + '/' + name;
  await writeFile(path.join(UPLOADS_DIR, rel), buffer);
  return '/uploads/' + rel;
}
```

`zagruzitPoURL(url, papka)` downloads from a URL first, then goes through the same path — used to persist Replicate output before it expires.


## Registration Rate-Limit by IP

To prevent bulk fake accounts, registrations are counted per IP per 24h:

```js
// api.js — POST /api/registraciya
const limReg = parseInt(getNastroyka('limit_reg_ip') || '1');
if (limReg > 0) {
  const ip = (req.headers['x-forwarded-for'] || req.connection.remoteAddress || '').split(',')[0].trim();
  const cutoff = new Date(Date.now() - 86400000).toISOString();
  const cnt = baza.prepare(
    'SELECT COUNT(*) as c FROM ip_registracii WHERE ip = ? AND sozdan > ?'
  ).get(ip, cutoff);
  if (cnt.c >= limReg) return res.status(429).json({ oshibka: 'Слишком много регистраций с вашего IP' });
  baza.prepare('INSERT INTO ip_registracii (ip) VALUES (?)').run(ip);
}
```

Limit is configurable from the admin panel, no redeploy needed.


## Maintenance Mode

Admin flips a flag in settings. All regular traffic gets a static maintenance page. Whitelisted IPs and active admin sessions pass through:

```js
// server.js
function techRezhimMiddleware(req, res, next) {
  if (req.path.startsWith('/api') || req.path.startsWith('/admin')) return next();

  const aktivno = baza.prepare(
    "SELECT znachenie FROM nastroyki WHERE klyuch = 'tech_rezhim'"
  ).get()?.znachenie;

  if (aktivno !== '1') return next();

  const ip = (req.headers['x-forwarded-for'] || req.connection.remoteAddress || '').split(',')[0].trim();
  const whitelist = (baza.prepare(
    "SELECT znachenie FROM nastroyki WHERE klyuch = 'tech_belyy_spisok'"
  ).get()?.znachenie || '').split('\n').map(s => s.trim()).filter(Boolean);

  if (whitelist.includes(ip) || req.session.admin) return next();
  res.sendFile(path.join(__dirname, '../frontend/pages/tech.html'));
}
```


## Gallery, Likes & Share Links

Likes are per-user, toggled via one endpoint:

```js
// api.js — POST /api/layk/:id
router.post('/layk/:id', trebuyetPolzovatelya, (req, res) => {
  const polzId = req.session.polzovatel.id;
  const genId  = parseInt(req.params.id);
  const sushch = baza.prepare('SELECT id FROM layki WHERE polzovatel_id = ? AND generaciya_id = ?').get(polzId, genId);
  const kol    = baza.prepare('SELECT COUNT(*) as c FROM layki WHERE generaciya_id = ?');

  if (sushch) {
    baza.prepare('DELETE FROM layki WHERE polzovatel_id = ? AND generaciya_id = ?').run(polzId, genId);
    return res.json({ liked: false, kol_laykov: kol.get(genId).c });
  }
  baza.prepare('INSERT INTO layki (polzovatel_id, generaciya_id) VALUES (?, ?)').run(polzId, genId);
  res.json({ liked: true, kol_laykov: kol.get(genId).c });
});
```

Share links: random 32-char hex token tied to the generation. Owners can share unpublished work:

```js
// api.js — POST /api/peredat/:id
const token = crypto.randomBytes(16).toString('hex');

const own = baza.prepare('SELECT * FROM generacii WHERE id = ? AND polzovatel_id = ?').get(genId, polzId);
if (own) {
  baza.prepare('INSERT OR REPLACE INTO share_tokeny (token, generaciya_id) VALUES (?, ?)').run(token, own.id);
  return res.json({ token, url: '/foto/' + token });
}
```


## Auto-Cleanup

Unpublished generations older than 3 days are purged on startup and every 24h — including cascade-deleting likes and share tokens:

```js
// server.js
function podchistitNeOpubl() {
  const cutoff = new Date(Date.now() - 3 * 24 * 60 * 60 * 1000).toISOString();
  const ids = baza.prepare(
    "SELECT id FROM generacii WHERE opublikovano = 0 AND sozdan < ?"
  ).all(cutoff).map(r => r.id);

  if (ids.length === 0) return;
  const ph = ids.map(() => '?').join(',');
  baza.prepare(`DELETE FROM layki        WHERE generaciya_id IN (${ph})`).run(...ids);
  baza.prepare(`DELETE FROM share_tokeny WHERE generaciya_id IN (${ph})`).run(...ids);
  baza.prepare(`DELETE FROM generacii    WHERE id            IN (${ph})`).run(...ids);
}

podchistitNeOpubl();
setInterval(podchistitNeOpubl, 24 * 60 * 60 * 1000);
```

## Production Practices Applied Across Projects

| Practice | Where |
|----------|-------|
| Idempotent webhook handling | UnitPay handler (checks existing payment before processing) |
| Database transactions | Payment credit + status update in BEGIN/COMMIT |
| IP whitelisting | UnitPay webhook (isUnitpayIp) |
| Rate limiting per IP | Registration limit (configurable from admin panel) |
| Graceful degradation | Cloudinary → local filesystem fallback |
| Maintenance mode + IP whitelist | techRezhimMiddleware |
| Configuration without redeploy | Admin panel editable settings table |
| Auto-cleanup (scheduled) | Unpublished generations purge every 24h |
| Caching (external API) | CBR exchange rate, 6h TTL |


---

# 2. Telegram Mass Sender - Multi-Account Automation Tool

A desktop application (CustomTkinter GUI) for managing multiple Telegram accounts and running parallel campaigns. Built as a full toolkit rather than a single script.

**Stack:** Python, Telethon, asyncio, CustomTkinter, OpenAI API (via OnlySQ)

**What's inside:**

- **Account manager** - add `.session` files, set limits, delays, and schedules per account
- **Mailing** - sends messages across a list of chats with randomized delays and repeat cycles
- **Infinite mailing (24/7 mode)** - each account distributes N messages per 24h, cooldown calculated automatically as `86400 / daily_limit ± 20%`
- **Group entry monitor** - detects new members joining specified groups, sends them a DM after a random delay
- **Keyword monitor** - reads all new messages in groups, matches against trigger rules, sends responses
- **Username watcher** - finds `@mentions` in group messages and writes to those users in DM
- **Parser** - extracts members by participants list or by message authors in a date range
- **Inviting** - adds parsed users to a target group, with per-account limits and rotation
- **Auto-reply** - uses GPT to answer incoming DMs naturally, one reply per user per session
- **Proxy support** - SOCKS5, HTTP, MTProto; distributed round-robin across accounts
- **Spam-ban detection** - catches `PeerFloodError`, checks `@SpamBot` for the unban date, logs it
- **Shift rotation** - accounts work on configurable day cycles (e.g. work 1 day, rest 3)
- **AI text variation** - rewrites outgoing messages via OnlySQ GPT before sending
- **Database** - JSON-based deduplication so the same user never gets messaged twice

```python
# Proxy parsing - supports raw ip:port, socks5://, and MTProto t.me/proxy links
def parse_proxy(raw: str):
    if "t.me/proxy" in raw:
        parsed = urlparse(raw)
        qs = parse_qs(parsed.query)
        return {"type": "mtproto", "server": qs["server"][0],
                "port": int(qs["port"][0]), "secret": qs["secret"][0]}
    if "://" in raw:
        parsed = urlparse(raw)
        return {"type": parsed.scheme, "server": parsed.hostname,
                "port": parsed.port, "user": parsed.username, "pass": parsed.password}
    parts = raw.split(":")
    return {"type": "socks5", "server": parts[0], "port": int(parts[1])}
```

```python
# Infinite mailing - even distribution across 24h with jitter
base_cooldown = 86400.0 / max(acc.infinite_daily_limit, 1)
jitter = random.uniform(-0.20, 0.20)
acc._inf_cooldown = round(base_cooldown * (1 + jitter), 2)
```

```python
# SpamBot check - reads the unban date from @SpamBot reply
await client.send_message("SpamBot", "/start")
await asyncio.sleep(5)
msgs = await client.get_messages("SpamBot", limit=3)
# parse date from the bot's response and set acc.banned_until
```

```python
# AI auto-reply - generates a short human-like response via GPT
def generate_auto_reply_sync(self, account, user_message):
    client = OpenAI(api_key=self.ai_api_key, base_url="https://api.onlysq.ru/ai/openai")
    prompt = f"{account.auto_reply_prompt}\n\nMessage: {user_message}\n\nShort reply:"
    response = client.chat.completions.create(
        model="gpt-5.2-chat",
        messages=[{"role": "user", "content": prompt}],
        max_tokens=100
    )
    return response.choices[0].message.content.strip()
```

---

# 3. Yandex Mail Mass Sender

A GUI tool for sending bulk emails via Yandex SMTP. Loads sender accounts from a file, rotates them randomly across recipients, supports multiple message templates, attachments, and retry logic for failed deliveries.

**Stack:** Python, smtplib, tkinter/ttk

**Key features:**
- Loads `login:password` accounts and recipient lists from `.txt` files
- Randomly picks a sender account for each recipient
- Multiple message templates separated by `###` - one is selected at random per send
- File attachments with Cyrillic filename support via `email.header.Header`
- Failed recipients saved to a timestamped file; retry button re-runs only those
- Settings (subject, body, delays, attachments) saved to JSON per OS convention (`AppData`, `~/.config`, etc.)
- Context menu and keyboard shortcuts (`Ctrl+C/V/X`) for all input fields

```python
# Random template selection and sender rotation
body = random.choice(templates)
random_account = random.choice(self.accounts)

# Attachment with Cyrillic filename
mime.add_header('Content-Disposition', 'attachment',
    filename=Header(filename, 'utf-8').encode())
```

> The script uses `smtplib` + `SMTP_SSL` on port 465. No third-party mail libraries.


---

# 4. Captcha Solver

A desktop tool that solves captchas automatically using a vision AI model. The user selects screen regions by pointing the mouse - captcha area, input field, reload button, skip button - and the solver runs in a loop.

**Stack:** Python, pyautogui, pynput, PIL, tkinter, OpenAI-compatible vision API (OnlySQ)

**How it works:**
1. Takes a screenshot of the selected region
2. Computes a pixel hash - if the image hasn't changed since last check, skips the API call
3. Sends the image to `c4ai-aya-vision-32b` as base64 with a system prompt that handles text captchas and math expressions
4. Types the answer character by character with random delays (`0.5–0.6s`)
5. Detects reload button by pixel color analysis (blue pixel count threshold)
6. Tracks duplicate answers, skip count, and reload attempts - auto-restarts after thresholds

```python
# Image hashing to skip unchanged captchas
def get_image_hash(image):
    small = image.resize((16, 16), Image.Resampling.NEAREST).convert('L')
    return hash(small.tobytes())

# Reload detection by pixel color
reload_blue = np.logical_and(
    np.logical_and(r < 30, g > 100),
    np.logical_and(g < 145, b > 240)
)
return np.count_nonzero(reload_blue) > 20
```

```python
# Answer typing with human-like delay
def type_answer(answer):
    for char in answer:
        keyboard_controller.type(char)
        time.sleep(random.uniform(TYPE_DELAY_MIN, TYPE_DELAY_MAX))
```


---

# 5. TGMansion - Telegram AI Bot

A Telegram bot that gives users access to GPT chat and image generation. Requires channel subscription to use. Includes admin broadcast functionality.

**Stack:** Python, aiogram 3, aiohttp, SQLite, OnlySQ API

**Features:**
- **Chat** - multi-model GPT chat (GPT-5.2, Qwen3 Max) with per-user message history, cooldown, and custom system role
- **Image generation** - Flux 2 Dev via OnlySQ Imagen API; optional prompt enhancement before generation
- **Prompt enhancement** - sends the user's prompt to GPT with a specialized system instruction, returns an enriched version
- **Subscription gate** - checks channel membership via `get_chat_member` before any action
- **Broadcast** - admin-only command to send a message/photo/document to all registered users
- **SQLite** - stores user IDs, usernames, join date; used for broadcast and stats

```python
# Subscription check before any handler
async def check_subscription(user_id: int) -> bool:
    chat_member = await bot.get_chat_member(chat_id=CHANNEL_USERNAME, user_id=user_id)
    return chat_member.status in ['member', 'administrator', 'creator']
```

```python
# Chat with message history and model rotation
chat_memory[user_id].append({"role": "user", "content": user_message})
response = await onlysq_chat_completion(
    model=model_id,
    messages=chat_memory[user_id]
)
ai_response = response["choices"][0]["message"]["content"].strip()
chat_memory[user_id].append({"role": "assistant", "content": ai_response})
```

```python
# Broadcast to all users with media support
for user in get_all_users():
    if message.photo:
        await bot.send_photo(user, message.photo[-1].file_id, caption=message.caption)
    elif message.text:
        await bot.send_message(user, message.text)
    await asyncio.sleep(0.1)  # rate limit
```


