# laskatel — Portfolio

Python developer. Working with Telegram APIs, AI integrations, automation, and web backends.

---

## Projects

### 1. Studio Myata — AI Photo Generation Platform

A full-stack web service built for a photo studio. Users can generate images via AI, browse a gallery, and purchase generation credits.

**Stack:** Node.js, Express, Vanilla JS, HTML/CSS

**Integrations:**
- **Replicate API** — image generation via Stable Diffusion and other models
- **UnitPay** — payment processing with webhook handling and order confirmation

**Key parts of the backend:**

```js
// unitpay.js — webhook from payment system
app.post('/unitpay/handler', (req, res) => {
  const { method, params } = req.body;
  if (method === 'pay' && verifySignature(params)) {
    creditsService.addCredits(params.account, params.sum);
    return res.json({ result: { message: 'OK' } });
  }
  res.json({ error: { message: 'Invalid request' } });
});
```

```js
// replicate.js — generation request
async function generateImage(prompt, model) {
  const output = await replicate.run(model, { input: { prompt } });
  return output[0]; // URL of the generated image
}
```

---

### 2. Telegram Mass Sender — Multi-Account Automation Tool

A desktop application (CustomTkinter GUI) for managing multiple Telegram accounts and running parallel campaigns. Built as a full toolkit rather than a single script.

**Stack:** Python, Telethon, asyncio, CustomTkinter, OpenAI API (via OnlySQ)

**What's inside:**

- **Account manager** — add `.session` files, set limits, delays, and schedules per account
- **Mailing** — sends messages across a list of chats with randomized delays and repeat cycles
- **Infinite mailing (24/7 mode)** — each account distributes N messages per 24h, cooldown calculated automatically as `86400 / daily_limit ± 20%`
- **Group entry monitor** — detects new members joining specified groups, sends them a DM after a random delay
- **Keyword monitor** — reads all new messages in groups, matches against trigger rules, sends responses
- **Username watcher** — finds `@mentions` in group messages and writes to those users in DM
- **Parser** — extracts members by participants list or by message authors in a date range
- **Inviting** — adds parsed users to a target group, with per-account limits and rotation
- **Auto-reply** — uses GPT to answer incoming DMs naturally, one reply per user per session
- **Proxy support** — SOCKS5, HTTP, MTProto; distributed round-robin across accounts
- **Spam-ban detection** — catches `PeerFloodError`, checks `@SpamBot` for the unban date, logs it
- **Shift rotation** — accounts work on configurable day cycles (e.g. work 1 day, rest 3)
- **AI text variation** — rewrites outgoing messages via OnlySQ GPT before sending
- **Database** — JSON-based deduplication so the same user never gets messaged twice

```python
# Proxy parsing — supports raw ip:port, socks5://, and MTProto t.me/proxy links
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
# Infinite mailing — even distribution across 24h with jitter
base_cooldown = 86400.0 / max(acc.infinite_daily_limit, 1)
jitter = random.uniform(-0.20, 0.20)
acc._inf_cooldown = round(base_cooldown * (1 + jitter), 2)
```

```python
# SpamBot check — reads the unban date from @SpamBot reply
await client.send_message("SpamBot", "/start")
await asyncio.sleep(5)
msgs = await client.get_messages("SpamBot", limit=3)
# parse date from the bot's response and set acc.banned_until
```

```python
# AI auto-reply — generates a short human-like response via GPT
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

### 3. Yandex Mail Mass Sender

A GUI tool for sending bulk emails via Yandex SMTP. Loads sender accounts from a file, rotates them randomly across recipients, supports multiple message templates, attachments, and retry logic for failed deliveries.

**Stack:** Python, smtplib, tkinter/ttk

**Key features:**
- Loads `login:password` accounts and recipient lists from `.txt` files
- Randomly picks a sender account for each recipient
- Multiple message templates separated by `###` — one is selected at random per send
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

### 4. Captcha Solver (bebe.py)

A desktop tool that solves captchas automatically using a vision AI model. The user selects screen regions by pointing the mouse — captcha area, input field, reload button, skip button — and the solver runs in a loop.

**Stack:** Python, pyautogui, pynput, PIL, tkinter, OpenAI-compatible vision API (OnlySQ)

**How it works:**
1. Takes a screenshot of the selected region
2. Computes a pixel hash — if the image hasn't changed since last check, skips the API call
3. Sends the image to `c4ai-aya-vision-32b` as base64 with a system prompt that handles text captchas and math expressions
4. Types the answer character by character with random delays (`0.5–0.6s`)
5. Detects reload button by pixel color analysis (blue pixel count threshold)
6. Tracks duplicate answers, skip count, and reload attempts — auto-restarts after thresholds

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

### 5. TGMansion — Telegram AI Bot

A Telegram bot that gives users access to GPT chat and image generation. Requires channel subscription to use. Includes admin broadcast functionality.

**Stack:** Python, aiogram 3, aiohttp, SQLite, OnlySQ API

**Features:**
- **Chat** — multi-model GPT chat (GPT-5.2, Qwen3 Max) with per-user message history, cooldown, and custom system role
- **Image generation** — Flux 2 Dev via OnlySQ Imagen API; optional prompt enhancement before generation
- **Prompt enhancement** — sends the user's prompt to GPT with a specialized system instruction, returns an enriched version
- **Subscription gate** — checks channel membership via `get_chat_member` before any action
- **Broadcast** — admin-only command to send a message/photo/document to all registered users
- **SQLite** — stores user IDs, usernames, join date; used for broadcast and stats

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


---

## Contact

Telegram: [@laskatel](https://t.me/laskatell)
