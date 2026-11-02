# PayFlow

### What actually happens in the two seconds after you tap "Pay"?

You press a button. A spinner turns. A green tick appears. Most of us never think about it again.

But behind that tick sit a DNS lookup, two separate handshakes, a token check, a firewall, a database write, a national payment switch and two different banks, all of which have to agree with each other, in order, over a network that may drop at any moment. Any one of them can fail. The interesting question is what the system does when they do.

**PayFlow is a browser toy that lets you watch all of it.**

On the left is a believable mock UPI app. On the right, every hidden step is spelled out as it happens, tagged by network layer, with the request and response text a real system would exchange, while a small packet travels along the route the payment would really take. Then you get to break things on purpose: drop the signal, tap twice, take a bank offline, or attack the request, and see why a well-built payment system survives each one.

> **Nothing here is real.** No money, no banks, no servers and no network calls. The whole thing is a single HTML file that plays out the story itself.

---

## Contents

- [Why this exists](#why-this-exists)
- [Who will get the most from it](#who-will-get-the-most-from-it)
- [Start in thirty seconds](#start-in-thirty-seconds)
- [A tour of the screen](#a-tour-of-the-screen)
- [The phone, screen by screen](#the-phone-screen-by-screen)
- [How to read the log](#how-to-read-the-log)
- [Break it: the experiments](#break-it-the-experiments)
- [Anatomy of one payment](#anatomy-of-one-payment)
- [The pretend API](#the-pretend-api)
- [The ideas, in plain words](#the-ideas-in-plain-words)
- [Controls](#controls)
- [Demo data and rules](#demo-data-and-rules)
- [A tour of the code](#a-tour-of-the-code)
- [Making it yours](#making-it-yours)
- [Design and accessibility](#design-and-accessibility)
- [Checking your work](#checking-your-work)
- [When something looks off](#when-something-looks-off)
- [Questions people ask](#questions-people-ask)
- [Words you'll meet](#words-youll-meet)
- [Honest limits](#honest-limits)
- [Where it could go next](#where-it-could-go-next)
- [Contributing, license and layout](#contributing-license-and-layout)

---

## Why this exists

Digital payments feel like magic because the machinery is hidden. That is good design for users and terrible for learning. A student reading about TCP handshakes, TLS or idempotency in a textbook has no way to *see* those things doing a job. A developer told "make payments retry-safe" has no concrete picture of what unsafe looks like.

PayFlow tries to close that gap with one rule: **show the invisible part, and show it failing.** Three promises sit behind that.

- **A double tap should never charge you twice.** You can watch the mechanism that stops it.
- **A dropped signal should never lose your money.** You can watch the app refuse to guess, ask politely for the truth, and get it.
- **A hostile request should never reach the database.** You can watch the layers that stop it, and see which layer caught which attack.

## Who will get the most from it

- **Students** of networking, web security or payment systems, who need to see concepts in motion rather than in diagrams.
- **Developers** who want a concrete mental model of retries, reversals and idempotency before they build the real thing.
- **Teachers and presenters** who need something clickable for a class, a demo or a viva, where each scenario maps to a single idea.
- **Product and design teams** exploring the user-facing states of a payment (pending, failed, refunded) and the wording that goes with each.

---

## Start in thirty seconds

1. Open `index.html` in any modern browser. Or, if you prefer a server, run `python3 -m http.server 8000` in the folder and visit `http://localhost:8000`.
2. On the phone, click **Skip with a demo account**.
3. Leave **Normal payment** selected in the scenario card.
4. Tap **Send money**, then **Continue**. The form is already filled in for you.
5. Type the PIN **1234** on the keypad.
6. Watch the packet move across the diagram and read the log as each step appears.
7. Press **Back to home**, choose a different scenario such as **Double tap**, and go again.

That is the happy path, and it takes less than a minute. Everything else in this document is about the *unhappy* paths, which is where the learning is.

**Want to go slowly?** Tick **Step through** before paying and press **Space** to reveal one event at a time. It is the best way to present this to other people, or to read a log you don't want to rush.

There is no build step and no dependency. The only outside request is to Google Fonts for IBM Plex; offline, the page quietly falls back to system fonts.

---

## A tour of the screen

| Where | What lives there |
|---|---|
| **Header** | The logo, the title, a reminder that none of this is real, and a **Reset demo** button |
| **Left column** | The phone. On wide screens it stays in view while you scroll the rest |
| **Top right** | **Path of a payment**: an animated diagram of the route a request takes |
| **Middle right** | **What happens, step by step**: the live event log and its controls |
| **Side cards** | The **Demo scenario** picker and the **Server and bank state** panel |

Below roughly 980 px the layout stacks into a single column, so it still works on a small laptop or a tablet.

### The route diagram

Six nodes make up the path:

**Phone → Internet → PayFlow server (port 443) → NPCI (the UPI switch) → Payer bank / Beneficiary bank**

A packet travels hop by hop, labelled with what it carries: `SYN`, `Hello`, `POST`, `ReqPay`, `Debit?`, `200 OK` and so on. Nodes light up as it passes through. Things can go wrong visibly:

- A packet can be **lost mid-route**, shown with a red ✕.
- When the signal drops, the **phone-to-internet link turns red and dashed**.
- In the tampering scenario an **attacker badge** appears on the internet node.

### The state panel

Three small instruments sit beside the log:

- **Counters:** *requests received* and *payments processed*. These two numbers are the quiet heroes of the double-tap experiment, because they are allowed to disagree.
- **Server database table:** the last five transactions with ID, payee, amount and status. You can watch a row appear as `INITIATED` and move through its states.
- **Bank ledgers:** live balances for the payer and the payee. A row flashes whenever its value changes. A payee gets a random starting balance the first time you pay them and keeps it afterwards.

---

## The phone, screen by screen

The mock app has a status bar with a live clock and a network indicator that flips from **5G** to **No signal** when the connection drops. Behind it are five screens.

**1. Register.** Full name, a 10-digit mobile number and a bank (SBI, HDFC, ICICI or Axis). The app derives a UPI address from your first name and bank, for example `arpit@sbi`. A note explains that the SIM is confirmed by SMS, which is called device binding. If you would rather skip all that, **Skip with a demo account** jumps to the home screen.

**2. Home.** A greeting, your payment address, your balance, a **Send money** button, and your last six payments with colour-coded status labels.

**3. Send money.** Who to pay, how much in rupees, and an optional remark of up to 40 characters. Errors appear inline as you type.

**4. Enter UPI PIN.** A custom four-digit keypad, with dots that fill in as you press. The demo PIN is `1234`; anything else is declined by the bank.

**5. Result.** A status icon, a title and a message, plus a details panel with the transaction ID, payee, amount, time and any note. **Back to home** appears only once the payment is truly final. A toast slides down on completion, and when the signal drops a "No internet connection" banner appears on this screen.

### The four states a payment can be in

`INITIATED` → `PENDING` → `SUCCESS` or `FAILED`

| State | Icon | What it means to the person holding the phone |
|---|---|---|
| `INITIATED` | Spinner | The app is opening a connection and sending the request |
| `PENDING` | Clock | Nobody knows the outcome yet. **Do not pay again** |
| `SUCCESS` | Tick | The money moved and the payee was credited |
| `FAILED` | Cross | The payment did not complete, and the message says whether money was never taken or was taken and returned |

One subtle thing is worth noticing early: **the phone and the server keep separate records.** There are moments when they disagree, for instance the server already says `SUCCESS` while the phone still shows `PENDING` after a dropped signal. That gap is exactly what makes payments hard, and exactly what the lost-reply experiment is about.

---

## How to read the log

Every event is a row with a coloured layer chip, a short title, a detail block in monospace, and a timestamp. The chip tells you *who is speaking*:

| Chip | The conversation it belongs to |
|---|---|
| **App** | HTTP requests and responses, server logic, database actions |
| **DNS** | Turning a name into an address, and caching the answer |
| **TLS** | Encryption, certificates and tamper detection |
| **TCP** | Opening a connection and delivering data reliably |
| **IP** | Addressing and routing between networks |
| **Link** | The radio or Wi-Fi hop, including losing signal |
| **UPI** | Messages between the server, NPCI and the banks |
| **Security** | Device binding, tokens, the firewall and rate limiting |

Rows turn **green** when something goes right (a connection opens, a reversal succeeds) and **red** when something fails or is attacked. Each run starts with a separator naming the scenario, so a long session stays easy to navigate.

Timestamps are theatre, not measurement. They creep forward by small random amounts per event, and jump by whole seconds where the app is genuinely waiting, such as before a retry.

---

## Break it: the experiments

Choose a scenario *before* you pay. The picker locks while a payment is running. Each experiment is designed to teach exactly one idea, so it helps to know what to look for before you start.

### 1. Normal payment: the control group

The full happy path. Select it, send ₹500 to the pre-filled payee with PIN `1234`, and follow the money.

**What to watch for**
- The connection set-up (DNS, TCP, TLS) before anything else happens.
- The `POST /api/v1/transactions` request that follows.
- The **Payer** ledger dropping by ₹500 and the **Payee** ledger rising by ₹500, at the debit and credit steps.
- The database row moving `INITIATED` → `PENDING` → `SUCCESS`.

### 2. Network drops after paying: the lost reply

This is the scenario most real apps get wrong. The bank has taken the money and the server has saved `SUCCESS`, but the reply never makes it back, and the phone loses signal.

A naive app would shrug, show "failed", and invite you to try again, which would charge you twice. PayFlow does something wiser: it admits it does not know.

**What to watch for**
- The phone shows **Payment pending** and tells you not to pay again.
- The first status check is lost. The signal returns, a *new* TCP connection opens, and TLS resumes from the earlier session instead of starting over.
- The retries wait longer each time (1 s, then 2 s). That is exponential backoff, and it stops a weak network being flooded.
- The retries are `GET` requests. They can only *read* the status, never re-send the payment.
- The second retry succeeds, and the payer was debited exactly once.

### 3. Double tap: the duplicate

The same request arrives a second time, carrying the same idempotency key.

**What to watch for**
- A row titled "Second tap: a duplicate POST arrives".
- **Requests received** goes up by two. **Payments processed** goes up by one.
- The payer balance is debited once.
- The result screen adds a note that you tapped twice.

The database refuses to insert a second row with the same transaction ID, so the duplicate is harmlessly ignored.

### 4. Beneficiary bank is down: the automatic undo

The payer's bank debits the money, but the receiving bank never answers.

**What to watch for**
- The credit request times out and the phone shows **Payment pending**.
- NPCI re-checks twice, then declares the payment `FAILED`.
- An automatic **reversal** puts the money back, and the payer balance returns to what it was.
- The result screen says the debited amount was returned.

The lesson: a system that can take money must also be able to give it back, without a human in the loop.

### 5. Attacker tampers with data: the altered request

An attacker sitting on the path flips bits in the encrypted request.

**What to watch for**
- The **attacker** badge appears on the internet node, and the log shows bits being flipped.
- The server rejects the record with a TLS `bad_record_mac` alert and closes the connection.
- **Requests received** does *not* increase, because the tampered request never reached the application at all.

The lesson: encryption is not only about secrecy. The authentication tag means any change at all is detected, even by someone who cannot read the data.

### 6. SQL injection attempt: the hostile input

The payee field carries a malicious payload, `x' OR '1'='1`, the way a hostile client would behave if it skipped the app's own format check.

**What to watch for**
- The malicious payee value appearing in the log.
- The firewall matching an injection rule and returning `403 Forbidden`.
- A request was received, but **no payment was processed and no database row was created**.

The lesson: the firewall is one layer, not the only one. Parameterised queries would protect the database even if the firewall missed it.

### Failures you get without a scenario

Normal payment is also a good place to cause ordinary trouble:

| What you do | What happens |
|---|---|
| Enter a PIN other than `1234` (try `0000`) | The payer bank declines it with **Incorrect UPI PIN**. Nothing is debited. The log notes that a real bank locks UPI after three wrong tries |
| Pay more than your balance (try over ₹25,000) | The payer bank declines it with **Insufficient balance**. Nothing is debited |
| Pay someone on an unknown bank (try `test@nobank`) | NPCI cannot find them: **Payee VPA not found**. Nothing is debited |

---

## Anatomy of one payment

Here is what a normal payment narrates, in order. Think of it as six chapters.

### Chapter 1: Getting connected (first payment only)

- A frame leaves the phone on the **link** layer.
- A **DNS** query goes out over UDP asking for the server's address. The answer is cached with a time-to-live so the next lookup is free.
- An **IP** packet is addressed to the server, passing through NAT, with a TTL and a route.
- A **TCP three-way handshake** runs: SYN, SYN-ACK, ACK.
- A **TLS 1.3 handshake** follows: ClientHello, then the server's certificate and key share, then Finished. The result is an encrypted tunnel using `TLS_AES_256_GCM_SHA384`.

### Chapter 2: The request

`POST /api/v1/transactions`, with a bearer token and an `Idempotency-Key` header. The log shows it in readable form, with a note that on the wire it is ciphertext.

### Chapter 3: The inspection

Before any money is considered, the server checks three things: the **firewall** (does this look like an attack?), the **token** (is the signature valid, has it expired?) and the **rate limit** (for example, payment 1 of 5 allowed this minute).

### Chapter 4: Writing it down

The transaction is inserted into the database as `INITIATED`, then updated to `PENDING`. Writing first, acting second, is what makes later recovery possible.

### Chapter 5: The trip to the switch

The server sends `ReqPay` to NPCI. NPCI looks up both banks. The payer bank verifies the PIN (carried as an encrypted blob that only that bank can open) and debits the account. The beneficiary bank credits the payee. NPCI returns `RespPay SUCCESS`.

### Chapter 6: Coming home

The server saves `SUCCESS` and replies `200 OK`. The phone updates. A push message and an SMS are also sent, so the person is told even if the app has been closed.

### Reusing the connection

If **Skip handshake after the first request** is ticked, later payments reuse the open connection (keep-alive) and skip DNS, TCP and TLS entirely. If it is not, the DNS answer is still reused from cache, but a full TCP and TLS handshake runs again each time. Toggling it is a quick way to feel what a handshake costs.

### A side story: registration

Registering runs the same connection set-up, then `POST /api/v1/auth/register`, then an SMS one-time code for **device binding** (shown as a toast, since there is no real SMS), then `POST /api/v1/auth/verify`. The server issues a **15-minute session token** and links your payment address to your bank account.

---

## The pretend API

These endpoints exist only as text and logic inside the page. They are here so the log reads like the real thing.

| Request | Purpose | Worth noticing |
|---|---|---|
| `POST /api/v1/auth/register` | Begin registration | Sends name, mobile number (`+91…`) and bank |
| `POST /api/v1/auth/verify` | Confirm the SMS code | Sends the one-time code and a device ID; returns a session token |
| `POST /api/v1/transactions` | Create a payment | Bearer token, an `Idempotency-Key` equal to the transaction ID, and a JSON body with payer, payee, amount, remark and the encrypted PIN |
| `GET /api/v1/transactions/{id}/status` | Ask how a payment went | Used after a lost reply or a pending result. It only reads, and never re-sends the payment |

**Responses you will see**

- `200 OK`: a final answer, which includes a payment that was declined.
- `202 Accepted`: received, but still pending.
- `403 Forbidden`: stopped by the firewall.

**The session token** is a JWT (`HS256`) whose payload holds your payment address and a 15-minute expiry. The page builds it itself, with a random signature segment.

**Transaction IDs** look like `PF1730381234`: `PF` followed by ten digits taken from the current time.

---

## The ideas, in plain words

Each of these is something you can point at on screen.

**Network layers.** A message does not travel as one thing. It is wrapped, layer by layer, for the radio link, for routing, for reliable delivery, for encryption and for the application. The log shows them in the order they really occur.

**DNS and caching.** Computers need numbers, people use names. DNS translates, over UDP because it is quick and small. The answer is cached for its TTL, which is why the second payment skips the lookup.

**TCP.** The three-way handshake is how two machines agree to talk, and "reliable, ordered delivery" is the promise that follows. It is the reason a lost packet gets resent without the app noticing.

**TLS 1.3.** Certificates prove who the server is. A key exchange produces a secret only the two ends know. AES-256-GCM encrypts the traffic and attaches a tag that catches tampering.

**Authentication.** Your identity travels as a signed token that expires quickly. Device binding, an SMS code tying the account to one SIM, makes a stolen account harder to use elsewhere.

**Request protection.** Input validation in the app, a firewall in front of the server, rate limiting on how often you may pay, and parameterised queries behind it. No single one is trusted to be perfect.

**Idempotency.** Doing something twice should have the same effect as doing it once. An idempotency key plus a uniqueness rule in the database makes retries and double taps safe.

**Failed versus unknown.** The most important distinction in the whole project. `FAILED` means we *know* it did not complete. `PENDING` means we *do not know*. A payment can be debited and pending at once, which is why the app says not to pay again.

**Polling and backoff.** When unsure, ask, do not repeat. And ask a little less eagerly each time.

**The shape of UPI.** Your server is not a bank. It talks to NPCI, which acts as the switch, and NPCI talks to two separate banks, one for each side of the money. Debit, credit, re-check and reversal are all messages between them.

**Safe failure.** Automatic reversal means a payment that fails never leaves the payer out of pocket.

**Defence in depth.** One layer failing should never expose everything. That is why three different things guard against bad input.

---

## Controls

| Control | What it does |
|---|---|
| **Slow / Normal / Fast** | Sets the pace of the animation and the log |
| **Step through** | Pauses at every event until you continue |
| **Next step**, **Space** or **→** | Advances one event in step-through mode. Ignored while you are typing in a text field |
| **Skip handshake after the first request** | Reuses the open connection for later payments |
| **Reset demo** | Clears all state and returns to the register screen |
| **Skip with a demo account** | Skips registration and signs in as `arpit@sbi` |

---

## Demo data and rules

**Fixed demo values**

- Demo user: `Arpit Kumar Jena`, address `arpit@sbi`
- Starting balance: **₹25,000**, restored on **Reset demo**
- Demo PIN: **1234**
- Registration and the send screen come pre-filled, so you can click straight through
- Payee balances are a random whole-hundred amount, created the first time each payee is paid

**Validation rules**

- **Payment address:** must look like `name@bank`: at least two characters of letters, numbers, `.`, `_` or `-` before the `@`, and letters after it.
- **Yourself:** you cannot pay your own address.
- **Amount:** a whole number, at least ₹1 and at most ₹1,00,000 per payment, which is the UPI limit.
- **Mobile number:** exactly 10 digits. **Name:** at least 2 characters.
- **Known bank handles:** `sbi`, `hdfcbank`, `icici`, `axisbank`, `ybl` and `paytm`. Any other handle is reported as not found.

**Illustrative only:** the server address, domain, token-lifetime text and firewall rule number in the log are examples, not real services.

---

## A tour of the code

Everything lives in `index.html`, in three parts: markup, one `<style>` block, and one `<script>` wrapped in a self-invoking function in strict mode.

**Markup.** The header, the phone with its five screens (`s-reg`, `s-home`, `s-send`, `s-pin`, `s-res`), the SVG route diagram, the log card, the scenario picker and the state card.

**Styles.** Colours are CSS variables on `:root`, with separate sets for light and dark. Components are small and flat: `.card`, `.phone`, `.screen`, `.btn`, `.pill` for status labels, `.chip` for layer labels and `.row` for log entries.

**Script.** One state object, `S`, is the single source of truth: speed, step mode, the user, balances, ledgers, transactions, connection flags, counters and the current transaction. Around it:

| Section | What it is responsible for |
|---|---|
| Helpers and constants | DOM shortcuts, `sleep`, random and hex helpers, rupee formatting, the `BANKS` table, layer names, `PACE` timings and the `SCEN` list |
| Pacing | `beat()` waits a fraction of the chosen pace, or waits for **Next step** in step mode. `advance()` releases it |
| Topology animation | `route()` finds a path through the diagram with a breadth-first search. `fly()` moves the packet hop by hop, and can drop it mid-route |
| Log | `addRow()` writes an entry. `E()` combines logging, packet movement and a pause. `sep()` adds a scenario separator |
| Phone UI | Switching screens, the toast, and renderers for the home list, result screen, database table and ledgers |
| Connection set-up | `connect()` runs the link, DNS, IP, TCP and TLS steps, or reuses the open connection when allowed |
| Registration | `register()` and `makeToken()` |
| Payment | `pay()` orchestrates a payment. `bankStage()` handles NPCI and the banks. `pollStatus()` handles retries. `finish()` and `failEarly()` set final results |
| Wiring | The keypad, form validation, scenario picker, controls, keyboard shortcut, clock, `resetDemo()` and `init()` |

The pleasant consequence of all this is that a run is just a long chain of `await`ed events. Adding a step means adding another `E(...)` call in the right place.

---

## Making it yours

### Small edits

| To change | Look for |
|---|---|
| Starting balance | `balance:25000` in the state object `S`, and again in `resetDemo()` |
| Demo PIN | The `txn.pin !== '1234'` check in `bankStage()`, and the hint text on the PIN screen |
| Per-payment limit | The `amt > 100000` check in `sendContinue()`, the `max` on the amount input, and the error message |
| Supported banks | The `BANKS` object. The registration dropdown keeps its own list |
| Animation speed | The `PACE` object, in milliseconds for slow, normal and fast |
| Scenario names and descriptions | The `SCEN` array |
| Diagram nodes and links | The `NODE` and `ADJ` objects, plus the matching SVG elements |
| Colours and theme | The CSS variables at the top of the `<style>` block |
| Demo account | The handler for **Skip with a demo account** in `init()` |

### Adding a scenario of your own

1. Add an entry to `SCEN` with an `id`, a title and a description.
2. Handle the new `id` in `pay()`, before or after the request, or in `bankStage()` if the trouble happens at a bank.
3. Emit each event with `E(layer, title, detail, [from, to, label], kind)`. Use a `kind` of `'good'` or `'bad'` to colour the row.
4. Finish with `setPhone(...)` and `finish(...)`, or with `failEarly(...)` if the payment never reaches the server.

Good candidates: an expired token, a slow server, a rate-limit hit, a certificate error.

---

## Design and accessibility

- **Themes.** Light and dark follow your system setting, each with a complete palette.
- **Layout.** Responsive, including safe-area handling for phones with notches.
- **Keyboard.** Visible focus rings, labelled keypad buttons, and Space or → for stepping.
- **Screen readers.** Status messages and errors use live regions, and the route diagram has a text description.
- **Motion.** The reduced-motion setting turns animations off.
- **Not by colour alone.** Every status has a text label and an icon as well as a colour.
- **Type.** IBM Plex Sans for text and IBM Plex Mono for code-style detail blocks.

**Browsers.** Current Chrome, Edge, Firefox and Safari. The page relies on CSS grid, custom properties, `:has()` (used to highlight the chosen scenario) and async functions, so very old browsers are not supported.

---

## Checking your work

There is no automated test suite, so here is a checklist for doing it by hand. Run it after any change.

**Registration and validation**
- [ ] Registering with the pre-filled values reaches the home screen
- [ ] A short name and a mobile number that is not 10 digits are both rejected
- [ ] **Skip with a demo account** signs in as `arpit@sbi` with ₹25,000
- [ ] A missing `@`, your own address, ₹0, decimals and amounts over ₹1,00,000 are all rejected

**Payments**
- [ ] Normal payment ends in `SUCCESS`, and both ledgers change by the right amount
- [ ] A wrong PIN ends in `FAILED` with no debit
- [ ] An amount above the balance ends in `FAILED` with **Insufficient balance**
- [ ] An unknown bank handle ends in `FAILED` with **Payee VPA not found**

**Scenarios**
- [ ] **Double tap** shows two requests received but one payment processed
- [ ] **Network drops after paying** shows pending, then success, with one debit
- [ ] **Beneficiary bank is down** ends in `FAILED` and the balance is restored
- [ ] **Attacker tampers with data** ends in `FAILED` and no database row is created
- [ ] **SQL injection attempt** ends in `FAILED` with `403 Forbidden` and no database row

**Controls and polish**
- [ ] Step-through pauses at each event, and Space and → advance it
- [ ] **Reset demo** restores the balance, counters, table and log
- [ ] Light and dark themes both look right

---

## When something looks off

| You see | Try this |
|---|---|
| Fonts look plain or different | The page could not reach Google Fonts and used system fonts. That is fine |
| Nothing happens after the PIN | Make sure you entered four digits. In **Step through** mode, press **Next step** or Space |
| The animation feels sluggish | Choose **Fast**, or turn **Step through** off |
| The scenario options are greyed out | A payment or registration is in progress. Wait for it to finish, or press **Reset demo** |
| Space does not advance the log | Step-through must be on, and your cursor must not be in a text field |
| Balances or counters look wrong after lots of testing | Press **Reset demo** for a clean slate |
| The page feels cramped | Widen the window. Below about 980 px everything stacks into one column |

---

## Questions people ask

**Is any of this real?**
No. There is no server, no bank and no network traffic beyond loading fonts. Every message in the log is generated by the page itself.

**Why is the PIN always 1234?**
It keeps the demo quick. It also gives you an easy way to trigger a declined payment on purpose: type anything else.

**Why does the PIN travel as an "encrypted blob"?**
In real UPI the PIN is encrypted on the device and only the issuing bank can read it. The prototype shows that idea in the log without implementing real cryptography.

**Why not simply retry a failed payment automatically?**
Re-sending a payment risks charging twice. PayFlow checks the status with a `GET`, which only reads, and uses an idempotency key so that even a duplicate `POST` cannot create a second payment.

**What is the difference between PENDING and FAILED?**
`PENDING` means the outcome is unknown. `FAILED` means it is known that the payment did not complete. A payment can be debited and pending at the same time, which is why the app tells you not to pay again.

**Can I use this in a class or a talk?**
Yes, that is a main use. Step-through mode and the three speeds are designed for presenting, and each scenario maps to one idea you can explain in a minute or two.

**Can I add my own scenario?**
Yes. See [Adding a scenario of your own](#adding-a-scenario-of-your-own).

---

## Words you'll meet

**UPI.** Unified Payments Interface: India's instant bank-to-bank payment system.

**VPA.** Virtual Payment Address, such as `name@bank`, used in place of an account number.

**NPCI.** The National Payments Corporation of India, which runs the UPI switch.

**Payer bank and beneficiary bank.** The bank the money leaves, and the bank it arrives at.

**ReqPay and RespPay.** The UPI request and response messages for a payment.

**Device binding.** Tying an account to one phone by sending a one-time code to its SIM.

**JWT.** JSON Web Token. A signed token that proves who you are for a short time.

**Idempotency key.** A unique ID sent with a request so that repeating it has the same effect as sending it once.

**DNS.** The system that turns a name like `api.payflow.in` into an IP address.

**TTL.** Time to live. For DNS, how long an answer may be cached. For IP, how many hops a packet may take.

**NAT.** Network address translation, where a router maps private addresses onto one public one.

**TCP handshake.** The SYN, SYN-ACK and ACK exchange that opens a reliable connection.

**TLS.** The protocol that encrypts and authenticates data between the app and the server.

**Keep-alive.** Reusing an open connection for several requests instead of opening a new one each time.

**Exponential backoff.** Waiting longer between each retry (here 1 s, then 2 s) so a weak network is not flooded.

**Reversal.** Returning a debited amount to the payer when the payment cannot be completed.

**Firewall (WAF).** A filter that blocks requests that look like attacks, such as SQL injection.

---

## Honest limits

- Every balance, bank and piece of data is simulated, and it all resets when you reload the page or press **Reset demo**.
- This is a teaching prototype. It is not production code and not a security reference. Real UPI involves many more checks, partners and regulations.
- The "encrypted" PIN and the TLS handshake are stories told in the log, not real cryptography.
- The one-time code appears as an on-screen toast because there is no real SMS.
- The rate limit, token-lifetime text and firewall rule number are fixed illustrative values, not live calculations.
- Log timestamps are simulated for readability and are not real measurements.

## Where it could go next

These are suggestions, not promises.

- A wrong-PIN counter that really locks UPI after three attempts
- A "request money" (collect) flow and a QR scan screen
- A transaction history screen with filters
- More ways to fail: slow server, expired token, rate-limit hit, certificate error
- An exportable log, as copied text or a download, for reports and write-ups
- A short guided tour that highlights each panel on first load
- Hindi and other Indian languages
- Automated tests for the validation and state logic

---

## Contributing, license and layout

**Contributing.** Ideas and fixes are welcome. Open an issue to talk through a change, or send a pull request. Please keep the project to a single file with no build step unless there is a strong reason, and run through the checklist above before submitting.

**License.** None has been chosen yet, so all rights are reserved by the authors for now. If you want others to be able to use or build on the prototype, add a `LICENSE` file (MIT and Apache-2.0 are common choices) and update this section.

**Layout.**

```
index.html   the entire prototype: HTML, CSS and JavaScript in one file
README.md    this file
```
