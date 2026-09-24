# Telegram AI Dəstək Bot — n8n + Ollama (Gemma4)

Bu workflow, Telegram vasitəsilə gələn dəstək mesajlarını lokal işləyən bir AI modeli (Gemma4, Ollama üzərindən) ilə avtomatik təsnif edir, müvafiq kateqoriyaya yönləndirir, cavab verir, Google Sheets-ə loglayır və xəta hallarını idarə edir.

## Ümumi Axın

```
Telegram Trigger
      │
      ▼
   If (mesaj mətndirmi?) ──false──► "Mətn yazın" cavabı
      │true
      ▼
HTTP Request (Ollama, Gemma4) ──error──► "Sistem məşğuldur" cavabı
      │success
      ▼
Code in JavaScript (JSON parse + validasiya)
      │
      ▼
   Switch (category)
      │
      ├── billing    → Telegram cavabı → Google Sheets loglama
      ├── technical  → Telegram cavabı → Google Sheets loglama
      ├── general    → If (hava sözü var?)
      │                   ├── true  → Open-Meteo API → Telegram (hava) → Sheets
      │                   └── false → Telegram (adi cavab) → Sheets
      └── Fallback   → Telegram ("başa düşmədik") cavabı → Google Sheets loglama
```

## Tələb olunan Mühit

| Komponent | Təyinat |
|---|---|
| **n8n** | Docker ilə self-hosted (`docker.n8n.io/n8nio/n8n`) |
| **Ollama** | Host maşında işləyir, model: `gemma4:e2b` |
| **ngrok** | Telegram webhook üçün public HTTPS tunel |
| **Telegram Bot** | BotFather vasitəsilə yaradılıb |
| **Google Sheets** | Loglama üçün, Service Account ilə qoşulur |

## Quraşdırma Addımları

### 1. Ollama
```bash
ollama pull gemma4:e2b
ollama serve   # arxa planda işləyir
```

### 2. ngrok
```bash
ngrok http 5678
```
Çıxan `Forwarding` ünvanını (`https://xxxx.ngrok-free.dev`) qeyd et.

### 3. n8n (Docker)
```bash
docker run -d --name n8n -p 5678:5678 \
  -e WEBHOOK_URL=https://xxxx.ngrok-free.dev/ \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```
**Diqqət:** n8n konteynerdə, Ollama isə host-da işlədiyi üçün, HTTP Request node-da Ollama ünvanı `http://host.docker.internal:11434/api/generate` olmalıdır — `localhost` işləməyəcək.

### 4. Telegram Bot
- BotFather-dan (`/newbot`) token al
- n8n-də Telegram credential-ına token-i yaz
- Workflow-u **Publish/Active** et ki, webhook qeydiyyatdan keçsin

### 5. Google Sheets (Service Account üsulu)
- Google Cloud Console-da Service Account yarat, JSON key endir
- Service Account email-ini Google Sheets sənədi ilə **Editor** icazəsi ilə paylaş
- n8n-də credential növü: **Service Account** (OAuth deyil — daha stabil, "sign in" tələb etmir)

## Node-ların Təyinatı

| Node | Rolu |
|---|---|
| **Telegram Trigger** | Gələn mesajları webhook ilə qəbul edir |
| **If (mətn yoxlaması)** | Sticker/şəkil kimi mətnsiz mesajları filtrləyir |
| **HTTP Request (Ollama)** | Mesajı Gemma4-ə göndərib JSON formatında kateqoriya alır: `{"category": "billing\|technical\|general"}` |
| **Code in JavaScript** | Ollama-nın JSON cavabını parse edir, yalnız icazəli 3 kateqoriyanı qəbul edir, əks halda `unrecognized` təyin edir |
| **Switch** | `category` sahəsinə görə 4 budağa yönləndirir |
| **If (hava sözü)** | `general` budağında, mesajda "hava" sözü olub-olmadığını yoxlayır |
| **HTTP Request (Open-Meteo)** | Bakı üçün canlı hava məlumatını çəkir (API açarı tələb etmir) |
| **Send a text message (×N)** | Hər budaq üçün fərqli, şablonlu Telegram cavabı |
| **Append row in sheet (×N)** | Hər tiketi Google Sheets-ə loglayır: Timestamp, User, Category, Message |

## Xəta İdarəetməsi

- **Boş/mətnsiz mesaj** → İlkin `If` node-u bunu tutur, Ollama-ya sorğu getmir
- **Ollama cavab verməsə (timeout/server dayanıb)** → HTTP Request node-un `Continue (using error output)` ayarı ilə ayrı bir xəta budağına yönləndirilir, istifadəçiyə nəzakətli mesaj göndərilir
- **AI-nin qaytardığı JSON pozulmuşsa/kateqoriya icazəli deyilsə** → Code node bunu tutub `unrecognized` təyin edir, Switch-in Fallback budağına yönləndirir

## Test Ssenariləri

| Mesaj | Gözlənilən Nəticə |
|---|---|
| "Faktura ilə bağlı sualım var" | `billing` budağı |
| "Sayt açılmır, xəta verir" | `technical` budağı |
| "Salam" | `general` — adi cavab |
| "Hava necədir bu gün?" | `general` — canlı hava məlumatı |
| Sticker/şəkil | "Mətn yazın" xəbərdarlığı |
| Ollama söndürülüb | "Sistem məşğuldur" cavabı |

## Bilinən Məhdudiyyətlər

- ngrok-un pulsuz planında hər yenidən başlatmada domen dəyişir — bu halda konteyner `WEBHOOK_URL` ilə yenidən yaradılmalı və workflow yenidən Publish edilməlidir
- Ollama modeli fəaliyyətsizlikdən sonra yaddaşdan boşalır (`keep_alive: "30m"` ilə bu, 30 dəqiqəyədək məhdudlaşdırılıb)
- Cold-start (ilk sorğu) 2-3 dəqiqə çəkə bilər, ona görə Timeout dəyəri yüksək (300000ms+) saxlanılmalıdır
