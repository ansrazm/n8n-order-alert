# n8n Order Alert Webhook + AI Summary

Workflow n8n yang menerima event order lewat webhook, memverifikasi signature
HMAC-SHA256, merangkum order memakai LLM (Claude API), lalu mengirim alert ke
Telegram. Request dengan signature salah ditolak dengan HTTP 401.

```
Webhook (POST) -> Verify Signature (HMAC) -> valid? -+-> AI Summary (Claude) -> Telegram -> 200
                                                     +-> 401
```

Konsep ini meniru alur webhook payment gateway di proyek Dragon Store
(verifikasi signature sebelum memproses event).

## Setup
1. Salin `.env.example` jadi `.env`, isi `WEBHOOK_SECRET`, `ANTHROPIC_API_KEY`, `TELEGRAM_CHAT_ID`.
2. `docker compose up -d`, buka http://localhost:5678 dan buat akun owner.
3. Import `workflow.json` (menu Workflows > Import from file).
4. Di node **Telegram Alert**, buat credential dengan token dari @BotFather.
   Kirim satu pesan ke bot-nya dulu supaya bisa mengirim ke chat kamu.
5. Klik **Execute workflow** (mode test), lalu jalankan curl di bawah.
   Untuk pemakaian terus-menerus, aktifkan workflow dan pakai URL `/webhook/order-alert`.

## Test
```bash
BODY='{"order_id":"DS-1042","customer":"budi","item":"Diamond 86","amount":25000,"status":"paid"}'
SIG=$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$WEBHOOK_SECRET" | awk '{print $2}')

# signature benar -> 200 + alert Telegram
curl -X POST http://localhost:5678/webhook-test/order-alert \
  -H "Content-Type: application/json" -H "x-signature: $SIG" -d "$BODY"

# signature salah -> 401
curl -i -X POST http://localhost:5678/webhook-test/order-alert \
  -H "Content-Type: application/json" -H "x-signature: salah" -d "$BODY"
```

Catatan: signature dihitung dari `JSON.stringify` body, jadi kirim JSON compact
(tanpa spasi) seperti contoh di atas.
