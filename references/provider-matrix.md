# Provider Capability Matrix

Verified by real e2e testing on 2026-07-14.

| Capability | DingTalk | Feishu | Notes |
|------------|:--------:|:------:|-------|
| P2P text | ✅ | ✅ | DingTalk: robot oToMessages/batchSend (batch of 20); Feishu: im/v1/messages |
| P2P markdown | ✅ native | ✅ via card | Feishu post rich-text does NOT accept standard markdown → rendered through interactive card `lark_md` |
| Group text / markdown | ✅ | ✅ | Both require the bot to be **in the group** first |
| Image | ✅ ≤20MB | ✅ ≤10MB | DingTalk media/upload → mediaId; Feishu im/v1/images → image_key |
| File | ✅ ≤20MB | ✅ ≤30MB | Feishu file_type inferred from extension (pdf/doc/xls/ppt/mp4/stream) |
| Mention person | ✅ md `<@userId>` | ✅ `<at user_id=...>` | |
| Mention all | ⚠️ textual | ✅ `<at user_id="all">` | DingTalk robot messages have no reliable native at-all; textual fallback |
| Recall | ✅ no time limit | ✅ within 24h | Feishu error 230020 = past time limit; card messages recallable same as text |
| Card messages | ✅ v1.1 ActionCard | ✅ v1.1 interactive | DingTalk: sampleActionCard(1 btn) / sampleActionCard2-5 (2-5 btns), 0 btn falls back to titled markdown; Feishu: header + lark_md + action buttons (no hard button limit) |
| Resolve by mobile | ✅ topapi getbymobile | ✅ batch_get_id | Requires respective contact permissions |
| Resolve by email | ❌ no API | ✅ batch_get_id | DingTalk has no email→userId endpoint; use alias instead |
| Dept lookup (builtin org) | ✅ listsub + listid | ✅ contact/v3 | Both need contact-read permissions |
| Create group | ✅ v2.0 dual-path | ✅ v2.0 im/v1/chats | DingTalk path A `--with-bot`: scene-group template (console-configured, robot bound in template) → bot in group, full robot channel. Path B `--no-bot`: legacy chat/create → bot NOT in group, messages auto-route to app-message channel chat/send (text/md/image/file OK, no buttons/recall). Ordinary DingTalk groups cannot add bots via API — manual only. Feishu: bot auto-joins, full capability; needs `im:chat` write scope |
| List groups | ❌ no API | ✅ im/v1/chats GET | DingTalk has no "groups the bot is in" API |

## Token model

| | DingTalk | Feishu |
|--|----------|--------|
| Endpoint | `POST /v1.0/oauth2/accessToken` | `POST /auth/v3/tenant_access_token/internal` |
| Credential | AppKey + AppSecret | App ID + App Secret |
| TTL | ~7200s | ~7200s |
| Cache | token_cache.json, refreshed 5 min early | same |
| Invalid-token retry | auto (once) | auto (once, codes 99991661/63/68) |

## Rate limits observed

- DingTalk batchSend: max 20 userIds per call → auto-chunked
- Both: 429/5xx get one backoff retry (see core/http.py)

## Known platform quirks

1. **Feishu open_id is per-app** — an open_id obtained under app A is invalid under app B. After switching apps, re-resolve users (aliases must be re-saved).
2. **Feishu permission changes require version release** — after granting scopes in the console you must create & publish a new app version.
3. **DingTalk Stream mode** — only relevant if you build a listener; multiple listeners on the same app compete for messages.
4. **DingTalk email** — no server-side email→userId lookup exists; guide users to mobile/userId/alias.
5. **Feishu accounts without email/mobile bound** cannot be resolved via batch_get_id; use group-member reverse lookup or alias.
