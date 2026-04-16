# Voice AI System Architecture — Tổng quan

> **Tầng 0 — Tổng quan** | Series: [Voice AI System Architecture](./README.md)
> ⏱ ~12 phút đọc · 💡 Sau bài này: hiểu toàn bộ bức tranh kiến trúc Voice AI production, biết tại sao 90% dự án thất bại — và map được 15 bài còn lại vào đúng vị trí trong hệ thống

---

Một startup healthcare đã demo AI receptionist cho nhà đầu tư — mượt mà, impressive. Latency 280ms, giọng tự nhiên, trả lời đúng context. Ba tuần sau, họ launch thật với Twilio qua số điện thoại thật.

Cuộc gọi đầu tiên: 2 giây im lặng sau khi bắt máy. User cúp máy.
Cuộc thứ hai: tiếng vỡ, AI nghe như robot bị hỏng.
Cuộc thứ ba: user đang nói bị AI ngắt lời và tiếp tục nói đè lên.

Họ đã dùng đúng GPT-4o Realtime. Prompt engineering chuẩn. Model không có vấn đề gì.

**Vấn đề là họ không biết G.711 là gì. Không biết ephemeral token. Không có barge-in state machine. Không có pre-connect.** Đây là kiến trúc — không phải AI.

---

## Tại sao đây là bài toán kiến trúc, không phải bài toán AI

> **Voice AI production = 20% model + 80% system engineering.**

Phần lớn tài liệu, tutorial, và YouTube video về Voice AI tập trung vào 20%: chọn model nào, prompt thế nào, fine-tune ra sao. Series này tập trung vào 80% còn lại — thứ quyết định liệu hệ thống có sống được ngoài môi trường production hay không.

Và 80% đó có thể định lượng được bằng tiền:

| | Cost/phút | Quy mô 10.000 cuộc/ngày |
|---|---|---|
| Human agent | $4–10 | $40.000–$100.000/ngày |
| AI (cascaded, GPT-4o) | $0.10–0.22 | $1.000–$2.200/ngày |
| AI (LLM + Neural Codec) | < $0.012 | < $120/ngày |

**Tiết kiệm 20–100x.** Nhưng chỉ khi hệ thống chạy được production. Demo không đếm.

---

## 5 lý do Voice AI chết ở production (không phải vì model)

Trước khi đi vào kiến trúc, cần hiểu đúng *loại* vấn đề mình đang giải:

**1. Audio codec không khớp**
User gọi từ điện thoại → audio đến server ở G.711 8kHz. AI model cần PCM 16kHz. Nếu conversion sai (dùng nearest-neighbor thay vì sinc interpolation) → tiếng vỡ, AI nhận diện sai. Đây không phải lỗi model — đây là lỗi audio engineering.

**2. Latency tích lũy**
STT 280ms + LLM 420ms + TTS 180ms + network 60ms = 940ms. Người dùng bắt đầu cảm thấy "chậm" ở 400ms. Mỗi bước tưởng "chấp nhận được" nhưng cộng lại vượt ngưỡng. Không có latency budget → không có cách debug.

**3. Session state mất khi disconnect**
User nghe vùng sóng yếu, call drop 3 giây rồi gọi lại. Nếu session state không được persist → AI không nhớ conversation trước, bắt đầu lại từ đầu. User phải giải thích lại toàn bộ.

**4. Barge-in không hoạt động**
AI đang đọc dài. User ngắt lời. AI không dừng — tiếp tục nói đè lên user. Trải nghiệm như nói chuyện với người điếc. Đây là lỗi state machine, không phải lỗi model.

**5. 2–3 giây im lặng sau khi bắt máy**
Server phải kết nối AI, load system prompt, generate lời chào → mất 2–3 giây. User nghĩ cuộc gọi bị lỗi hoặc scam → cúp máy trước khi AI kịp nói câu đầu tiên.

**Model không giải quyết được bất kỳ vấn đề nào trong số này.** System architecture mới giải quyết được.

---

## Kiến trúc tổng thể — 2 Channel, 1 Core

Hệ thống Voice AI production chia làm **4 tầng**, với nguyên tắc cốt lõi: **2 channel đầu vào hoàn toàn khác nhau — nhưng hội tụ về cùng 1 AI Core.**

```
┌─────────────────────────────────────────────────────────┐
│ CLIENT LAYER                                            │
│  ┌──────────────────┐    ┌──────────────────┐           │
│  │  🌐 Browser      │    │  📞 Phone        │           │
│  │  WebRTC · Opus   │    │  PSTN · G.711    │           │
│  │  48 kHz          │    │  8 kHz (Twilio)  │           │
│  └────────┬─────────┘    └────────┬─────────┘           │
└───────────┼───────────────────────┼─────────────────────┘
            ▼                       ▼
┌─────────────────────────────────────────────────────────┐
│ GATEWAY LAYER                                           │
│  ┌──────────────────┐    ┌──────────────────┐           │
│  │  Token API       │    │  WSS Stream      │           │
│  │  ephemeral key   │    │  Twilio/Telnyx   │           │
│  │  30min TTL       │    │  audio bridge    │           │
│  └──────────────────┘    └──────────────────┘           │
└──────────────────────────────┬──────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────┐
│ AI CORE  ← Tất cả channel hội tụ về đây                 │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Real-time Voice Model                            │  │
│  │  PCM 16kHz in → PCM 24kHz out · sub-300ms         │  │
│  └───────────────────────────────────────────────────┘  │
│  Template Service · Session Service · Observability     │
└──────────────────────────────┬──────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────┐
│ DATA LAYER                                              │
│  Security (PII masking · Guardrails)                    │
│  Redis Cache (bridge state · semantic cache)            │
│  PostgreSQL (sessions · transcripts)                    │
└─────────────────────────────────────────────────────────┘
```

**Insight cốt lõi: Channel chỉ là adapter — Core mới là bộ não.** Browser và Phone là 2 thế giới hoàn toàn khác nhau về codec, protocol, và latency profile. Nhưng cả hai đều phải được chuẩn hóa về PCM trước khi AI Core xử lý. Phần lớn bug production nằm ở lớp adapter này — không phải ở Core.

---

## Audio Pipeline — Nguyên tắc bất đối xứng

Thành phần mà hầu hết tutorial bỏ qua hoàn toàn.

```
INBOUND (Phone → AI) — có thể buffer
  Phone       Decode      Upsample      Buffer      AI Model
  G.711 8kHz → Linear PCM → 16kHz    → ~60ms   →  PCM 16kHz
                                       [imperceptible]

OUTBOUND (AI → Phone) — zero buffer
  AI Model    Low-pass    Downsample    Encode      Phone
  PCM 24kHz → anti-alias → 8kHz     → G.711    →  8kHz
                                       [immediate]
```

**Inbound — buffer được:** 60ms nằm dưới ngưỡng perception (~80ms). Gom đủ chunk trước khi xử lý giúp model chính xác hơn.

**Outbound — zero tolerance:** Bất kỳ delay nào ở output đều cảm nhận được ngay lập tức. Buffer output = tiếng nói bị "giật".

> ⚠️ **Lỗi phổ biến:** Apply cùng buffer strategy cho cả 2 chiều. Kết quả là AI nghe "lag" dù metrics P95 vẫn ổn — vì buffer outbound không xuất hiện trong latency measurement.

---

## Ephemeral Token — Tại sao API key không được vào browser

Với channel browser, có một vấn đề bảo mật cơ bản: **WebSocket cần kết nối thẳng đến AI endpoint — nhưng không thể đặt API key trong client code vì bất kỳ ai cũng đọc được.**

**Pattern giải quyết bằng 2 plane tách biệt:**

```
CONTROL PLANE — Lấy token (qua HTTP server)

  Browser ──[POST /api/live-token]──→ Server ──→ AI Endpoint
          ←──[ephemeral token, 30min TTL]────────
  (API key giữ lại ở server, không bao giờ ra client)

DATA PLANE — Stream audio (trực tiếp, KHÔNG qua server)

  Browser ◄══[WSS + ephemeral token]══► AI Model
           ◄─── Audio stream (bidirectional) ───►
          (zero server latency · real-time)
```

Kết quả:
- **Server không bao giờ chạm vào audio** → zero latency phía server cho data path
- **API key không bao giờ ra khỏi server** → client chỉ thấy token tạm thời

Đây là pattern tách **control plane** khỏi **data plane** — cùng cách Twilio, Daily.co, LiveKit thiết kế. → Chi tiết: [Bài 11 — Ephemeral Token & Security](./11-ephemeral-token-security.md)

---

## Pre-Connect — Giải quyết vấn đề số 5

Nhớ lại vấn đề thứ 5: 2–3 giây im lặng sau khi bắt máy. Pre-Connect Pattern giải quyết bằng cách **làm việc trong lúc điện thoại đang reo**.

```
❌ KHÔNG CÓ PRE-CONNECT
  Bắt máy → Connect AI (800ms) → Load prompt (400ms) → Greeting (1200ms)
             └─────────────── 2-3 giây im lặng ──────────────────────┘

✅ CÓ PRE-CONNECT
  Phone reo → Connect AI (800ms) ┐
            → Load prompt (400ms) ├ song song với tiếng reo
            → Cache greeting     ┘

  User bắt máy → Play greeting (100ms) ✓ Conversation starts immediately
```

**Không phải model nhanh hơn. Là làm sớm hơn.** → Chi tiết: [Bài 13 — Optimization Patterns](./13-optimization-patterns.md)

---

## 4 Kiến trúc AI Pipeline

Đây là quyết định kiến trúc quan trọng nhất — và không có "đúng/sai" tuyệt đối, chỉ có trade-off.

| Kiến trúc | Latency | Cost/phút | Flow | Phù hợp |
|-----------|---------|-----------|------|---------|
| **Cascaded** | 600ms–1.5s | $0.10–0.22 | STT → LLM → TTS | Healthcare, Legal, Enterprise |
| **Speech-Native** | 160–400ms | $0.06–0.15 | Single model | Consumer apps, UX-first |
| **Thinker-Talker** | ~257ms | $0.08–0.18 | Thinker → Talker | New projects 2025+ |
| **LLM + Neural Codec** | <50ms stream | <$0.012 | LLM → Neural Codec | High-volume, self-hosted |

| | Cascaded | Speech-Native | Thinker-Talker | Neural Codec |
|---|---|---|---|---|
| Latency | ●●○○ | ●●●● | ●●●○ | ●●●● |
| Control | ●●●● | ●●○○ | ●●●○ | ●●○○ |
| Cost | ●●○○ | ●●●○ | ●●○○ | ●●●● |
| Maturity | ●●●● | ●●●○ | ●●○○ | ●○○○ |

→ So sánh chi tiết: [Bài 07 — 4 AI Pipeline Architectures](./07-ai-pipeline-architectures.md)

---

## Các Pattern Kiến trúc Production

### Unified Server — Anti-microservices có chủ đích

Gom HTTP + WebSocket + audio bridge vào cùng 1 process. Nghe "sai" theo convention microservices — nhưng đúng cho Voice AI.

**Lý do:** Audio bridge truy cập session state mỗi ~20ms. Nếu tách microservices, mỗi operation cần 1 network round-trip → 20ms/hop × nhiều hops = latency budget vỡ ngay. In-memory access là ~0.1µs. Network access là ~0.5ms. **Khác nhau 5.000 lần.**

### Speculative Tool Calling — Che latency bằng parallelism

Khi AI nhận diện được user sắp cần một action (flight booking), thay vì chờ user nói xong rồi gọi API, AI **bắt đầu gọi API song song** trong khi user vẫn đang nói.

```
User nói: "Tôi muốn đặt vé máy bay từ Hà Nội..."
                    ↓
AI nhận diện intent: BOOKING_FLIGHT
→ Bắt đầu gọi flight API NGAY (không chờ user nói xong)
                    ↓
User tiếp: "...đến TP.HCM ngày mai" [API vẫn đang chạy]
                    ↓
User nói xong: "Tìm kiếm cho tôi"
               └─ API đã có kết quả sẵn
                    ↓
AI respond ngay: "Tôi thấy có 3 chuyến..." ✓ Near-zero tool latency
```

**Kết quả:** Latency bị "che" bằng user's speaking time. User không cảm thấy bất kỳ delay nào.

### Barge-in State Machine — User có quyền ngắt lời

Hệ thống phải có state machine để xử lý khi user ngắt lời AI:

```
  LISTENING ──[user nói]──→ PROCESSING ──[xong]──→ SPEAKING
      ▲                                                │
      │                                                │
      └────────── VAD: user nói lại ───────────────────┘
                  (Cancel TTS ngay lập tức)
```

> ⚠️ **Lỗi phổ biến:** Xử lý barge-in bằng cách chờ AI nói xong rồi mới process interrupt. User đã cúp máy lâu rồi.

### Template-Driven System

Không hardcode conversation flow trong code. Persona, prompt, flow config đều là dữ liệu trong Template Store — thay đổi behavior của AI mà không cần deploy lại code.

→ Chi tiết: Bài 12 — Unified Server Pattern · Bài 13 — Optimization Patterns (sắp ra)

---

## Tóm tắt — Bức tranh tổng thể

Nhìn lại toàn bộ bài viết: Audio Bridge, Ephemeral Token, Pre-Connect, Barge-in State Machine, Unified Server — **không cái nào liên quan đến model AI**. Tất cả đều là system engineering thuần túy.

Đây là lý do 90% dự án Voice AI chết ở production: team build demo xong nghĩ "chỉ cần thêm Twilio là xong". Thực tế, Twilio chỉ là 1 dòng trong danh sách 20 thành phần cần xử lý đúng.

**Series này giải quyết 20 thành phần đó — từng bài một, từng gốc lên.**

---

## Đọc tiếp — Chọn path phù hợp với bạn

**Bạn mới với Voice AI** → Bắt đầu tuần tự từ Tầng 1:
→ **[Bài 01 — Audio Fundamentals](./01-audio-fundamentals.md)** · Hiểu PCM, sample rate, G.711, Opus — nền tảng để đọc mọi thứ còn lại

**Bạn đang debug production** → Đọc 3 bài này trước:
→ Bài 04 — Audio Bridge · Bài 06 — Latency Budget · Bài 14 — Observability (sắp ra)

**Bạn đang chọn kiến trúc cho dự án mới** → 2 bài quyết định:
→ Bài 07 — 4 AI Pipeline Architectures · Bài 15 — Cost Analysis (sắp ra)

**Toàn bộ roadmap 15 bài** → [Voice AI System Architecture — README](./README.md)

---

*Tổng quan (0/15) · [Bài 01 — Audio Fundamentals →](./01-audio-fundamentals.md) · [Roadmap](./README.md)*
