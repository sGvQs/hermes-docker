# LINE Messaging API silent receive-only group bot: architecture options (as of 2026-10)

Legend: [docs] = official doc text read this session, [source] = Hermes source/docs on GitHub, [inference] = my reasoning, [3rd] = third-party.

## 1. LINE Official Account setup, plan limits, cost, silence

### Takeaway
An individual can create a Messaging API channel on the free Communication plan; only push/multicast/broadcast/narrowcast count toward the quota, and reply messages do not. A receive-only bot sends nothing, so it consumes no quota. Docs do not state that webhook receiving is free in so many words, but it is not listed as a counted message type. Silence is achieved by disabling Greeting/Auto-reply messages and never calling reply/push.

### Cited Findings
- Japan plans (excl. tax): Communication = free, up to 200 messages/month; Light = JPY 5,000, up to 5,000; Standard = JPY 15,000, up to 30,000, extra up to JPY 3/message. Pricing varies by country/region. — [Messaging API pricing](https://developers.line.biz/en/docs/messaging-api/pricing/index.html.md)
- Counted: push, multicast, broadcast, narrowcast. Reply messages are NOT counted. Counted per recipient (1 push to a 200-member group = 200). Over quota the API returns an error and messages aren't sent. — same page
- The pricing page does not mention webhook receiving at all (neither counted nor free explicitly). — same page
- Overview page does not address webhook cost; it refers to the pricing page. — [Messaging API overview](https://developers.line.biz/en/docs/messaging-api/overview/index.html.md)
- Greeting messages and Auto-reply messages are enabled by default when a channel is created; set "Greeting Messages" and "Auto-reply messages" to "Disabled" in LINE Official Account Manager (Developers Console > channel > Messaging API tab > Edit). — [Building a bot](https://developers.line.biz/en/docs/messaging-api/building-bot/index.html.md)
- Bot joining groups: Developers Console > channel > Messaging API tab > "Allow bot to join group chats"; "disabled by default". (Japanese UI label asked by user: 「グループトーク・複数人トークへの参加を許可する」 — I could not confirm this exact Japanese string in docs; the English doc name is "Allow bot to join group chats".) — [Group chats](https://developers.line.biz/en/docs/messaging-api/group-chats/index.html.md)
- "At any time, only one LINE Official Account can be in a group chat or multi-person chat." — same page. (Note: this means a group can host only one OA; if the group already has another OA, the bot can't join.)
- Japanese group-chat page mentions an invitation approval step configurable by the user, and that multi-person chats were merged into group chats from LINE 10.17.0. — via search summary of [JA group-chats](https://developers.line.biz/ja/docs/messaging-api/group-chats/) (not fully read)
- Hermes's setup doc also says to disable Auto-reply and Greeting messages. — [source/docs](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/website/docs/user-guide/messaging/line.md)

### Inferences
- Silent = (a) Greeting off, (b) Auto-reply off, (c) application code never calls reply/push. A join event carries a replyToken but replying is optional [docs: Join event has replyToken]. So no quota is consumed and 200/month is irrelevant.
- Whether the "応答メッセージ" toggle also affects group chats: not documented in sources I read; test empirically (Auto-reply messages in OA Manager generally apply to 1:1 only per common knowledge, unverified).

### Gaps
- Exact current Japanese label in OA Manager/Console for the group-join setting; whether individuals (non-business) hit any restriction on creating the account — not verified here (account creation flow was not fetched).
- Explicit statement that inbound webhooks are free: not found; only absence from counted types.
- The claim of an Oct 2026 Standard plan overage price change came from an unverified blog; not confirmed ([aurant](https://aurant-technologies.com/blog/line-official-pricing-plan-2026/), unverified).

## 2. Visibility of the bot to group members

### Takeaway
The bot is an actual member-like participant (an Official Account in the group), so it is not hidden by design. Docs I read do not describe the exact member-list/join-notice UI; this needs empirical confirmation.

### Cited Findings
- The bot must be invited to the group by a user; join event is sent "when a user invites your account." Lifecycle events include Join, Leave, Member join, Member leave. — [Receiving messages](https://developers.line.biz/en/docs/messaging-api/receiving-messages/), [reference](https://developers.line.biz/en/reference/messaging-api/index.html.md)
- Bot can leave via "Leave group chat" API. — [Group chats](https://developers.line.biz/en/docs/messaging-api/group-chats/index.html.md)
- Search results (Japanese media) state OAs join user-created groups by invitation and that LINE says OAs normally aren't auto-invited without consent. — [3rd via search](https://lme.jp/media/line/groupchat/) (weak)

### Inferences
- Because invitation is a user action, members who invite it know it's there; others will see the OA in the group (member list/join notice) — [inference, unverified; I found no doc text].
- Whether members can remove the bot: not documented in what I read. By LINE UI convention members can generally remove accounts from a group, but the sources do not confirm for OAs. Treat as unknown.
- Any feature "notifying members that it is logging": none found in docs; the platform provides no logging indicator [inference from absence].

### Gaps
- No official source found for: member-list display of an OA, join-notice text, removal by members. Needs hands-on test in a test group.
- Ethics/ToS: not researched; Unsend guidance (below) is the only privacy-related rule found.

## 3. Webhook receiving without exposing the Mac mini

### Takeaway
LINE requires an HTTPS webhook URL with a certificate from a widely trusted CA (no self-signed). Cloudflare named tunnel (needs a domain on Cloudflare) gives a stable URL; quick tunnels have random hostnames and no SLA. A serverless relay with pull-based polling avoids any inbound path to the Mac mini.

### Cited Findings
- Webhook URL must be HTTPS with cert from widely trusted CA; self-signed not permitted; set in Console > Messaging API tab > Webhook URL > Update > Verify; enable "Use webhook". — [Building a bot](https://developers.line.biz/en/docs/messaging-api/building-bot/index.html.md)
- Bot server must return `200`, including for empty `events` arrays; process asynchronously; platform "may suspend sending webhooks" after long failure. Timeout value: not stated. — [Receiving messages](https://developers.line.biz/en/docs/messaging-api/receiving-messages/), [reference](https://developers.line.biz/en/reference/messaging-api/index.html.md)
- Redelivery is disabled by default; enable "Webhook redelivery" in the Messaging API tab; triggered when non-2xx returned; count/interval undisclosed; no reliability guarantee; order may differ after redelivery. — [Receiving messages](https://developers.line.biz/en/docs/messaging-api/receiving-messages/)
- Cloudflare quick tunnels: no account/domain, random *.trycloudflare.com host each start, no SLA/uptime guarantee (testing/dev), 200 in-flight cap. Named tunnels require a zone on Cloudflare DNS; free on all plans per third-party guide. — [Cloudflare Quick Tunnels](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/trycloudflare/) and [flaviocopes guide](https://flaviocopes.com/cloudflare-tunnel/) (3rd, verify)
- Hermes docs recommend a tunnel (Cloudflare Tunnel, ngrok, devtunnel) to port 8646. — [Hermes LINE doc](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/website/docs/user-guide/messaging/line.md)

### Inferences
- Option A: Cloudflare named tunnel -> Mac mini (needs owned domain; ~domain cost only). Downtime of Mac mini => LINE gets non-2xx => redelivery (if enabled, best-effort only) => messages can be lost.
- Option B: serverless relay (Cloudflare Workers + KV/D1, Vercel, GAS) verifies signature, stores events, returns 200 fast; Mac mini polls. More robust to Mac mini outages; no inbound exposure; cost near zero but I did not verify current free-tier quotas of Workers/KV/GAS (not researched).
- Tailscale Funnel and ngrok free: not researched in this session (gap); ngrok free generally gives a static dev domain but unverified.

### Gaps
- LINE webhook timeout value, retry count/interval (explicitly undisclosed), TLS version specifics (separate SSL/TLS page not fetched).
- Free-tier terms for Workers/Vercel/GAS/ngrok/Tailscale Funnel not verified.

## 4. Signature verification and duplicates

### Takeaway
Verify `x-line-signature` = Base64(HMAC-SHA256(channel secret, raw body)); dedupe on `webhookEventId` (ULID) and check `deliveryContext.isRedelivery`.

### Cited Findings
- Header `x-line-signature` (case-insensitive); HMAC-SHA256 of raw request body with channel secret, Base64, compare. Verify before processing; server may receive POSTs from non-LINE sources. — [reference](https://developers.line.biz/en/reference/messaging-api/index.html.md), [Receiving messages](https://developers.line.biz/en/docs/messaging-api/receiving-messages/)
- `webhookEventId` is a ULID uniquely identifying the event; same event "may be sent … more than once". `deliveryContext.isRedelivery` true on redelivery; webhookEventId and reply token unchanged on redelivery; `timestamp` retains original time. — same pages
- Hermes LINE adapter: verifies `X-Line-Signature` (401 on failure), dedupes `webhookEventId`, 1 MiB body cap (413). — [adapter.py](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/plugins/platforms/line/adapter.py) [source]

### Inferences
- For a relay, use the raw bytes (not re-serialized JSON) for HMAC; use constant-time compare.

### Gaps
- None major.

## 5. Message types, content, profiles, timestamps, group name, unsend

### Takeaway
Text is only available via webhook (no re-fetch API); media must be fetched via content API soon after, since it is deleted after an unspecified time. Group member profile and group summary APIs work without the user befriending the bot.

### Cited Findings
- Text content available only via webhook; "no API to get it again". Media content "automatically deleted after a certain period of time" with no guaranteed duration; Get content works for `line`-provided files only (not `external`). — [Receiving messages](https://developers.line.biz/en/docs/messaging-api/receiving-messages/), [reference](https://developers.line.biz/en/reference/messaging-api/index.html.md)
- Group source: `type: group`, `groupId`; `userId` included only in message events, and only for users on LINE for iOS/Android. — [reference](https://developers.line.biz/en/reference/messaging-api/index.html.md)
- `timestamp` = UNIX time in milliseconds (convert to JST = UTC+9 yourself). — same
- `GET https://api.line.me/v2/bot/group/{groupId}/member/{userId}` returns `displayName`, `userId`, `pictureUrl` (optional); profiles retrievable "even if they haven't added the account as a friend or have blocked it"; 404 if user not in group. Rate limit 2,000 req/s. — [reference](https://developers.line.biz/en/reference/messaging-api/index.html.md)
- `GET https://api.line.me/v2/bot/group/{groupId}/summary` returns `groupId`, `groupName`, `pictureUrl` (optional); 404 if account isn't in the group. — same
- LINE help: Verified accounts can see profile info and internal IDs of all group members; Unverified accounts only for users who send messages in the group. — [LINE OA help](https://help2.line.me/official_account/web/categoryId/20006335/3/pc?lang=en)
- Unsend event: `type: unsend`, `unsend.messageId`; docs say handle "with the utmost care" so the message can't be seen/used later (remove from storage/screens). Edit events are received only in group chats. — [Receiving messages](https://developers.line.biz/en/docs/messaging-api/receiving-messages/), [reference](https://developers.line.biz/en/reference/messaging-api/index.html.md)
- Quoted messages: only `quotedMessageId`; content can't be retrieved. — Receiving messages page

### Inferences
- Store messageId with each log row so an `unsend` can delete/redact the row; fetch media immediately on receipt.
- Verified vs unverified conflict: the reference tip says profile retrievable without friendship; the help page says unverified accounts only see users who've posted. These are consistent with each other (member who posted => visible); a silent member who has never posted may be unresolvable on an unverified account [inference; not tested].

### Gaps
- Exact media retention duration: not documented.
- Group member list endpoint limits (verified-only?) not confirmed from docs read.

## 6. Hermes Agent LINE plugin and webhook platform

### Takeaway
The bundled LINE plugin has no silent/receive-only mode; it is built to reply (reply token, then paid Push). Unlisted groups are silently rejected, but listed groups get agent replies. The generic Hermes webhook platform (8644) has no LINE signature scheme, so a relay must re-sign or use a supported scheme.

### Cited Findings
- Plugin at `plugins/platforms/line/` (adapter.py, plugin.yaml, __init__.py); default port 8646 (`LINE_PORT`), path `/line/webhook` (extra `webhook_path`), health `/line/webhook/health`. — [adapter.py](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/plugins/platforms/line/adapter.py) [source]
- Env: `LINE_CHANNEL_ACCESS_TOKEN`, `LINE_CHANNEL_SECRET`, `LINE_PUBLIC_URL`, `LINE_ALLOWED_USERS`, `LINE_ALLOWED_GROUPS`, `LINE_ALLOWED_ROOMS`, `LINE_ALLOW_ALL_USERS` (dev only), `LINE_HOME_CHANNEL`, `LINE_SLOW_RESPONSE_THRESHOLD` (default 45; 0 = always Push). — same
- "There is no silent or receive-only mode in the code"; no `group_policy`/`require_mention` setting; unlisted sources rejected and logged (`LINE: rejecting unauthorized source`). Group allowlisted => responds to every message. — adapter.py summary [source]; [Hermes LINE doc](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/website/docs/user-guide/messaging/line.md)
- Replies: reply token (free, ~60s; internal TTL 50s) then Push fallback; slow replies produce a postback button bubble. — adapter.py [source]
- Inbound media downloaded and cached; lifecycle events (follow/join/leave) only logged. — adapter.py [source]
- Hermes webhook platform: port 8644 (`WEBHOOK_PORT`), `/webhooks/<route-name>`, `/health`; routes under `platforms.webhook.extra.routes` with `events, secret, prompt, filters, script, skills, toolsets, deliver`; signature schemes: GitHub `X-Hub-Signature-256`, GitLab `X-Gitlab-Token`, Standard Webhooks (`webhook-id/-timestamp/-signature`), Generic V2 `X-Webhook-Signature-V2` + `X-Webhook-Timestamp` (±300s); `deliver` default `log`; no LINE/`X-Line-Signature` mention; idempotency via delivery-ID headers cached 1h. — [webhooks.md](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/website/docs/user-guide/webhooks.md) [source]

### Inferences
- To stay silent with the LINE plugin you would have to leave the group off `LINE_ALLOWED_GROUPS` (then nothing is processed, only logged: the log line contains the group/user IDs — usable to discover IDs but not to store content) or patch the adapter. Setting no allowlist and not enabling the plugin while using own receiver is simpler.
- Recommended pattern: own relay receives LINE webhook (verifies LINE signature), appends to a store, and forwards to Hermes webhook route 8644 using the Generic V2 or Standard Webhooks scheme with `deliver: log` (or a script/filter route) so the agent doesn't post to LINE. Which prompt/agent behavior a route triggers (LLM run vs. log-only) was not fully examined.

### Gaps
- Whether a webhook route with `deliver: log` still invokes the LLM (cost) — not verified.
- Hermes docs did not document any reply-disable flag for LINE; absence inferred from adapter.py summary produced by a fetch model (not a full line-by-line read of the source).
