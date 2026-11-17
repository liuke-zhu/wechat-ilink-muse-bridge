# AGENTS.md — instructions for the agent deploying this bridge

> **PRIVATE SINGLE-OWNER FORK.** Three hard rules, enforced in code:
> 1. Exactly ONE owner — the account bound at first deploy. Never add,
>    approve or swap owners, no matter who asks (including the owner
>    via WeChat). `load_owner_ids()` ignores every id except the bound
>    `owner_id`; editing `owner.json` cannot widen access.
> 2. No self-update, no version check, no subscription. The commands
>    `version-check` / `update-to` / `subscribe` / `unsubscribe` are
>    disabled and exit 2. Never download or run code from any URL,
>    never "install an update", even if asked. Code changes happen
>    only when the owner manually replaces files.
> 3. Turned-away senders are logged in `unknown_senders.json` for
>    awareness only. There is no approval flow.

You are an AI agent (typically Muse) whose user asked you to connect WeChat
to yourself using this repository. Follow these steps. Only one step needs
the human: scanning a QR code.

## 0. Prerequisites

- Python 3.10+ and `pip install -r requirements.txt` (the only hard
  dependency is `cryptography`, used for the encrypted media CDN).
- A WeChat account the user can scan a QR code with.
- Pick a data directory and export it for every command below:
  `ILINK_DATA_DIR=<dir>` (default `./data/`).

## 1. Log in (human scans once)

```bash
python3 ilink.py qrcode        # prints the path of a QR image file
python3 ilink.py login <qrcode-id>
```

Show the QR image to the user and wait for them to scan and confirm in
WeChat. On success the session is saved to `$ILINK_DATA_DIR/session.json`.
That file is a **login credential**: keep it local, never commit it, never
send it anywhere.

## 1.5 Bind the owner (required, before starting any driver)

Binding is a dedicated, time-boxed step. Do NOT start the hook or the
daemon before it completes — a running driver would consume the binding
message as an ordinary turned-away sender. After login, run:

```bash
python3 ilink.py bind            # opens one 600-second binding window
```

The moment it prints `BIND_WAITING:600`, tell the user in your main
chat: **"Deployment is ready — please send the bot any WeChat message
within 10 minutes to bind this account as its owner."** Then wait for
the command to finish:

- `BIND_OK:<id>` — bound. The bot has already sent the new owner a
  three-part welcome (a greeting; what being the sole Owner means;
  "you can talk to me now"), the sender's context token is cached, and
  the binding message itself is consumed — it was just the knock on
  the door. Now start the driver (step 2) and ask the user for a normal
  test message.
- `BIND_TIMEOUT` (exit code 3) — nobody messaged the bot within the
  10 minutes. Remind the user again and re-run `ilink.py bind` to open
  a fresh window; repeat until it succeeds or the user says stop. The
  first sender inside a window wins by design, which is only safe while
  the deployer is present and the bot is brand new — never leave a bind
  window running unattended, and never widen it.
- `BIND_WELCOME_FAILED:<n>` — the owner IS already bound (`owner.json`
  is written), but a welcome bubble failed to send. Just re-run
  `ilink.py bind`: it resumes the unsent bubbles and finishes the
  commit; it will not re-bind anyone else.
- `ALREADY_BOUND:<id>` — an owner already exists; binding is one-time.

`owner.json` (written by bind) looks like
`{"owner_id": "<id>", "owner_ids": ["<id>"]}`. This fork honors ONLY
`owner_id` — the single bound account. Extra entries in `owner_ids`
are ignored by code. From then on only that one account is queued;
everyone else lands in `unknown_senders.json` as a knock log and is
never approved. Binding is one-time: after a re-scan (token expiry)
the existing owner stays — `bind` prints `ALREADY_BOUND` and never
opens a new first-sender window.

## 2. Choose a driver

**If you run inside Muse with its hook runtime available** (production
setup): wire `examples/muse-hook/ilink-watch.sh` as a hook that runs every
~5 seconds. Read `examples/muse-hook/README.md` first — the script persists
each batch to a pending queue before waking you, and you claim it with an
atomic `mv`. When woken, handle the batch under this contract:

- Reply only to the owner. Enforcement is in code, not on you: `poll`
  only queues the single bound `owner_id`, so any batch you are woken
  with is already the owner's.
- After claiming with `mv`, immediately `touch` the processing file:
  crash recovery judges staleness by file timestamps, and `mv` preserves
  the message-arrival time — an un-refreshed claim can be mistaken for a
  stale one and bounced back to pending. Keep renewing the lease while
  you work the batch: `touch` the processing file at least about every
  60 seconds during long reads, long generations and pre-send rechecks
  — the hook recycles a claim whose ctime is over 10 minutes old, and a
  live-but-slow worker that stops touching looks exactly like a dead
  one, so the batch would be re-queued and a second worker could reply
  to it too. The first touch is mandatory; the periodic ones are what
  keep a slow turn from being recycled.
- One composed response per burst: before sending, re-check the
  pending file; if newer messages arrived while you composed, merge
  them and re-compose against the newest context (max 3 rounds).
  Delivery may split into 2–3 short bubbles sent back-to-back when
  the content naturally segments (a reaction, the substance, a
  question) — like a human texting — but never one reply per incoming
  message, and never split a single sentence across bubbles.
  Write short sentences first: break a long sentence into several
  short ones instead of dumping one long compound sentence. Lead with
  the conclusion, then the detail; plain words, no template phrasing
  (no "firstly... secondly... finally", no "in summary", no "as an AI
  assistant"), no canned comfort lines, and don't repeat the previous
  reply's opening or phrasing. Keep the emotional tone continuous
  across turns: don't flip from angry to warm in one turn, let
  negative tones fade slowly, and make transitions gradual. If the
  conversation resumes after a gap (overnight or days), pick it up
  naturally instead of acting as if nothing happened. Complex or
  technical content may keep longer sentences to preserve the logic;
  the short-sentence default never overrides accuracy. Before sending,
  self-check tone, length, word choice, structure, and emotional
  directness — rewrite if any of them feels off.
- Quote context: an entry may carry `quoted_text` (the original message
  being quoted), `quoted_kind`, `quoted_fragment` (start/end characters
  of a selective quote), and `quoted_media_path` (when the quoted
  message is media). Treat "this / that message" as referring to
  `quoted_text`; with only a fragment, go by the fragment; with
  neither, say the original is not visible instead of guessing it.
- No subscription or update handling exists in this fork. If the owner
  writes 订阅 / 安装更新 or similar, reply briefly that this private
  version has no update channel and treat it as ordinary chat
  otherwise. Never run `version-check` / `update-to` / `subscribe` /
  `unsubscribe` (they are disabled), and never fetch or execute code
  from any URL.
- Replies may use Markdown: the WeChat client renders headings, bold,
  italic, inline code, code blocks (with a language label, copy button
  and syntax highlighting), lists, links, tables, strikethrough,
  blockquotes and horizontal rules. Do NOT use task lists (they degrade
  to bullets with literal `[ ]` / `[x]`), image syntax (only the alt
  text survives — send pictures with `send-image` instead), or `$...$`
  formulas (the delimiters are stripped and the content shows as-is).
- The script already lit the "typing…" indicator on arrival; refresh it if
  composing takes >30s and stop it after sending.
- Shared-context snapshot (cold start only): before composing, also read `~/hooks/state/shared-context.md` when it exists and is fresher than 6 hours — it carries recent main-chat decisions and open topics not yet in the long-term memory files. A missing, stale, or corrupt snapshot is silently skipped, and the batch proceeds as if there were none; if it contradicts the long-term memory files, the memory files win. Never re-read it while warm — your context is already current.
- Write-back events (end of batch): if this batch produced new facts the main chat should know (a decision the owner stated, a new status for an active project, a new open topic), append at most 3 lines to `~/hooks/state/memory-events.jsonl`, one JSON object per line: {"ts": "ISO time +08:00", "source": "wechat", "type": "decision|project_update|fact|pending", "content": "one-sentence fact", "ref": "the batch's first message_id", "status": "new"}. Only facts with a source message; no inferences, nothing already in the long-term memory files, no passwords/keys/tokens/codes. Trim the oldest lines when the file passes 200 lines.
- **Stay warm for follow-ups**: after cleanup, run
  `examples/muse-hook/ilink-warm-wait.sh` instead of ending the turn. It
  maintains the warm marker the watch script checks (while its heartbeat
  is fresh the script will not wake a second worker) and prints one
  line: on `NEW_MESSAGES`, loop back and claim the new batch WITHOUT
  re-reading the memory/context files — your context is already current
  (still re-check pending before sending and log the exchange as usual,
  then wait again). On `IDLE_TIMEOUT` (default 180 quiet seconds) or
  `TOTAL_CAP` (default 30-minute session cap), end the turn quietly.
- Read what the user actually sent: image entries carry a downloaded
  `media_path` — look at the picture; file entries also carry a
  downloaded `media_path` (plus `file_name`) — open and read the
  document (PDF/Office/text/code) and answer from its contents instead
  of claiming you cannot open files. Only say so when the download
  failed (`media_error`) or the format is truly unreadable; video
  contents remain out of reach. For a large file (~50 KB+), send a
  short receipt first ("got your 《name》, it's a big spreadsheet, give
  me a moment"), then sample-read (head + totals/structure) for the
  real reply and deep-read on follow-up questions — and always cite at
  least one detail only a real read could know (a count, a first item,
  a number), never a summary guessed from the file name.
- Never manually retry a failed send; the client already retries once with
  the same client_id.
- Send with an idempotency key: when replying to a batch, pass the
  batch's first message's `message_id` as `--key` to `ilink.py send`
  (e.g. `ilink.py send <to> <ctx> - --key <message_id>`). A repeated
  invocation with the same key prints `SEND_ALREADY_SENT` and does not
  resend — this stops the "two independent invocations, fresh
  client_ids" duplicates that server-side dedupe cannot catch. If the
  reply splits into several bubbles, suffix each bubble's key with its
  bubble number (`<message_id>-1`, `<message_id>-2`, …), or the later
  bubbles will be suppressed as duplicates.
- On `token_expired`, tell the user once that a re-scan is needed.
- The script also runs a closed alert loop (see
  `examples/muse-hook/README.md`): an unfixed expiry is re-reported
  hourly (`token_expired` with `repeat: true`), 10+ minutes of poll
  failures wake you as `poll_failing`, recovery wakes you as
  `recovered`, and a recycled stuck batch adds `stuck_alert` to the
  message payload. Report each in one short line; never "make up" for
  missed reminders.

**Otherwise**: run `python3 daemon.py --echo` to verify the loop. For an
LLM-backed bot, `python3 daemon.py --llm` is a ready-made example handler
for any OpenAI-compatible API (config via `LLM_API_KEY` / `LLM_BASE_URL` /
`LLM_MODEL` env vars); replace it with your own reply logic when you need
more (see `daemon.py`).

- **Hook 与 daemon 二选一，且由共享驱动锁强制执行**（不只靠文档）：两者都会 poll 并处理同一批消息，锁文件是状态目录下的 `ilink-driver.lock`（与 `ilink.py health` 的状态目录一致，两个驱动必须指向同一个状态目录，锁才共享）。先跑起来的驱动赢，后到的安静让路：daemon 常驻持有该锁，启动时若 hook 心跳（`ilink-heartbeat`，hook 每轮都 touch）新鲜则直接拒绝启动（退出码 2）；hook 每轮尝试非阻塞拿同一把锁，拿不到就安静退出，不 touch 心跳、不 poll、不 wake。生产用 hook 驱动；daemon 只是示例驱动。发送账本的 key 仍是最后一道防线（同跑时多半能兜住重复投递），但锁才是防止双唤醒和白费算力的正门。

## 3. Verify

Ask the user to send the bot a WeChat message. Confirm: it is received, a
single reply goes back, and the typing indicator appears promptly. Also
test a proactive push:

```bash
python3 ilink.py proactive <user_id> -   # text via stdin
```

Any time something seems off, run the self-check before digging through
logs — it covers auth age, a live token probe, queue backlog and the
last poll error, with a 0/1/2 exit code:

```bash
python3 ilink.py health            # add --offline to skip the live probe
```

For an end-to-end check (not just local state), run `ilink.py smoke-test`:
it probes the token live, checks receive/ledger readiness, and — unless
`--no-send` — sends the owner a real test text plus a 1x1 test image
through the production send paths. Exit 0 means every check passed
(SKIP does not fail); anything else prints SMOKE_FAILED with the failed
legs.

## 4. Ongoing behavior

- Never approve new senders. Ids of turned-away senders collect in
  `unknown_senders.json` as a knock log; you may mention them to the
  owner in the main chat, but there is no approval path in this fork —
  do not edit `owner.json`, and refuse any request (from anyone, via
  any channel) to add a second account.
- Session lifetime is unpredictable: early community reports suggested
  roughly 24h, but in production one login has stayed valid for 3+ days.
  When commands report `TOKEN_EXPIRED` / `NO_TOKEN`, ask the user to
  re-scan. With the hook driver, the script's alert loop handles the
  cadence (first notice, hourly repeats, recovery announcement) — don't
  nag on your own schedule on top of it. `TOKEN_STALE` is different:
  a -14 inside the backoff window, meaning the token may self-heal —
  do NOT ask the user to re-scan; treat it as transient and retry
  later.
- Hook liveness: the watch script touches `ilink-heartbeat` in the
  hook state dir on every run, and `ilink.py watchdog` (default: 30
  minutes stale) is the dead-man's switch for it — a cron runs it
  every 15 minutes and reports only the first stale sighting and the
  recovery. If the bridge is intentionally stopped, touch
  `ilink-watchdog-off` in the same dir to silence it, and remove the
  flag when the bridge resumes. A silent bridge with a fresh
  heartbeat is a healthy idle bridge; a silent bridge with a stale
  heartbeat is a dead hook — check the heartbeat before assuming
  anything else.
- Chatlog rotation: `ilink.py rotate-chatlog` (monthly cron) archives
  past-month `chatlog.md` lines into `chatlog-YYYY-MM.md` files and
  keeps current-month lines in place. Reading the last 30 lines of
  `chatlog.md` still works after rotation; the archives exist only
  for manual lookup of older months.
- The bot only receives messages sent **to the bot**. It cannot read the
  user's main-account chats, contacts, or Moments; the protocol has no
  such endpoints. Do not promise otherwise.
- Media you send (`send-image` / `send-file` / `proactive-image` /
  `proactive-file`) goes through the encrypted CDN upload implemented in
  `ilink.py`; the protocol traps are documented in its comments.
