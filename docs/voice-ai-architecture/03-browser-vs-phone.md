# Bài 03 — Browser vs Phone: 2 Thế Giới Kết Nối

> **Tầng 1 — Nền tảng** | Series: [Voice AI System Architecture](./README.md)
> ⏱ ~12 phút đọc · 💡 Sau bài này: hiểu tại sao browser và phone tạo ra hai "thế giới" hoàn toàn khác nhau về codec, latency, kiến trúc — và tại sao AI Core phải được thiết kế independent khỏi cả hai

---

## Bài toán cốt lõi — Một AI Core, hai channel hoàn toàn khác nhau

Nếu bạn xây Voice AI system, khả năng cao sẽ có 2 hướng kết nối:

1. **Browser/Web App** — User nói vào microphone → WebRTC → AI → TTS → Speaker
2. **Điện thoại thực** — User gọi số điện thoại → PSTN → Twilio WebSocket → AI → TTS → lại điện thoại

Cả hai phải đến cùng một AI Model. Nhưng đây không chỉ là codec khác nhau — đó là **hai thế giới** về audio quality, latency, kiến trúc xử lý, và failure modes. Hiểu rõ sự khác biệt này là điều kiện tiên quyết để thiết kế kiến trúc đúng.

---

## Thế giới 1: PSTN — mạng điện thoại 50 tuổi vẫn thống trị

**PSTN (Public Switched Telephone Network)** là hạ tầng điện thoại toàn cầu, số hóa từ thập niên 1960. Khi ai đó gọi đến số hotline công ty bạn, cuộc gọi đó đi qua PSTN.

**Cách PSTN hoạt động:** Điện thoại chuyển giọng nói thành tín hiệu điện, gửi đến tổng đài (Central Office). Tổng đài số hóa thành G.711 (8kHz, 8-bit) rồi chuyển qua mạng lõi bằng **circuit switching** — một "đường dây ảo" được dành riêng cho cuộc gọi từ đầu đến cuối, băng thông cố định 64kbps bất kể nội dung (im lặng cũng tốn 64kbps).

Trong bối cảnh Voice AI, cuộc gọi PSTN **không đến server trực tiếp**. Nó đi qua CPaaS như Twilio hoặc Telnyx — chi tiết ở mục sau.

**Đặc điểm kỹ thuật PSTN channel:**

| Thông số | Giá trị | Ý nghĩa |
|----------|---------|---------|
| Codec | G.711 µ-law/A-law (cố định) | Không có lựa chọn thay thế |
| Sample rate | 8 kHz | Tái tạo tần số đến 4 kHz — giọng "mỏng" |
| Bitrate | 64 kbps (cố định) | Im lặng cũng tiêu thụ bandwidth |
| Latency transport | 30-80ms (PSTN) + 10-30ms (gateway) | Tổng 80-200ms chỉ riêng truyền tải |
| Reliability | 99.999% uptime | Không bị ảnh hưởng bởi mất internet |

**Về chất lượng audio:** 8kHz chỉ tái tạo được tần số đến 4kHz. Âm sibilant ("s", "f", "th") bị cắt, giọng nghe "mỏng" và kém tự nhiên. Đây là giới hạn cứng của hạ tầng PSTN — không thể cải thiện dù băng thông internet ngày nay rẻ như nước.

---

## Thế giới 2: WebRTC — stack hoàn chỉnh sinh ra cho realtime

**WebRTC (Web Real-Time Communication)** là bộ giao thức và API cho phép truyền audio/video realtime trực tiếp giữa browser và server. Google mở mã nguồn năm 2011, W3C/IETF chuẩn hóa, mọi browser lớn hỗ trợ native.

**WebRTC không chỉ là giao thức — nó là một stack hoàn chỉnh:**

**Tầng Audio:** Tích hợp sẵn Acoustic Echo Cancellation (AEC) — loại bỏ tiếng vọng khi loa phát lại audio mà mic thu được. Thêm Automatic Gain Control (AGC) tự động điều chỉnh âm lượng, và Noise Suppression (NS) lọc tiếng ồn nền (quạt, xe cộ, bàn phím). **Ba tính năng này miễn phí và tự động** — PSTN không có.

**Tầng Codec:** Opus là codec mặc định — adaptive 6-510kbps, hỗ trợ 8-48kHz, thiết kế đặc biệt cho speech realtime. Khi im lặng, DTX (Discontinuous Transmission) tiêu thụ gần 0 bandwidth. Chất lượng vượt trội G.711 ở mọi bitrate.

**Tầng Transport:** SRTP (Secure Real-time Transport Protocol) chạy trên **UDP** — khác hoàn toàn WebSocket (TCP). Packet bị mất thì bỏ qua, không retry. Nghe "tệ" nhưng là thiết kế đúng: nhận packet âm thanh trễ 200ms thì phát lại cũng vô nghĩa, thà bỏ qua còn hơn chờ. DTLS thêm mã hóa end-to-end bắt buộc.

**Tầng Network:** ICE + STUN + TURN framework — tự động tìm đường kết nối tốt nhất qua NAT và firewall. STUN tìm IP public, TURN làm relay khi direct connection không thể (~10-15% trường hợp). Xử lý tự động, developer không cần config.

**Đặc điểm kỹ thuật WebRTC channel:**

| Thông số | Giá trị | Ý nghĩa |
|----------|---------|---------|
| Codec | Opus 48kHz (adaptive) | Chất lượng HD, tự điều chỉnh theo mạng |
| Sample rate | 48 kHz | Gấp 6 lần PSTN, tái tạo đầy đủ dải tần |
| Bitrate | 6-510 kbps (adaptive) | Mạng yếu giảm bitrate, không drop call |
| Latency transport | 20-80ms | Ultra-low latency, direct connection |
| Reliability | Phụ thuộc internet | Opus FEC + jitter buffer xử lý packet loss <5% |

---

## Twilio và Telnyx — cầu nối PSTN vào thế giới internet

Server Voice AI không kết nối trực tiếp với mạng PSTN. Các nhà cung cấp **CPaaS (Communication Platform as a Service)** như Twilio và Telnyx đóng vai trò cầu nối:

**Cách hoạt động:**

```
Phone (PSTN)
    → Twilio/Telnyx (nhận cuộc gọi từ PSTN)
    → WebSocket connection đến server của bạn
    → Stream audio G.711 dưới dạng JSON messages (base64, 20ms/chunk)
    ← Server gửi audio response ngược lại qua cùng WebSocket
```

Mỗi JSON message từ Twilio chứa: audio payload (base64-encoded), sequence number, timestamp. Server không nhận raw audio — nhận JSON wrapper, cần decode base64 → G.711 → PCM trước khi xử lý.

**Twilio** cung cấp: SIP Trunk, phone number provisioning, Media Streams (WebSocket audio), TwiML (call flow scripting). Pricing theo phút + phí số điện thoại hàng tháng.

**Telnyx** tương tự nhưng thường rẻ hơn cho high-volume, có private IP network (ít hop hơn, latency thấp hơn), API gần tương thích với Twilio.

> **Production note:** Nhiều system tích hợp cả Twilio và Telnyx để **failover** — nếu một provider có sự cố, tự động chuyển sang provider còn lại. Đây là thực hành quan trọng với uptime SLA 99.99%+.

---

## So sánh trực diện — tại sao kiến trúc phải khác nhau

| Tiêu chí | Browser (WebRTC) | Phone (PSTN) |
|----------|------------------|--------------|
| **Codec** | Opus (negotiate tự động) | G.711 µ-law (cố định) |
| **Sample rate** | 48 kHz | 8 kHz |
| **Bitrate** | Adaptive 6-510 kbps | Cố định 64 kbps |
| **Echo cancel** | AEC tích hợp sẵn | Server phải tự xử lý |
| **Noise suppression** | Tích hợp sẵn | Server phải tự xử lý |
| **Server xử lý audio?** | Không (direct) | Bắt buộc (transcode) |
| **Latency transport** | 20-80ms | 80-200ms |
| **Reliability** | Phụ thuộc internet | 99.999% uptime |
| **Chất lượng voice** | HD — rõ, ấm, tự nhiên | SD — mỏng, hẹp tần số |
| **Scale model** | Scale theo signaling | Scale theo compute (CPU) |
| **Failure mode** | Mất internet → reconnect | Server chết → all calls drop |

**Audio quality gap ảnh hưởng trực tiếp đến STT accuracy:** AI model nhận input 16kHz. PSTN phải upsample 8→16kHz (bổ sung thông tin không tồn tại — quality bị giới hạn). WebRTC downsample 48→16kHz (bỏ bớt thông tin — quality vẫn tốt). Cùng model nhưng WER (Word Error Rate) khác nhau rõ rệt giữa hai channel.

**Latency budget không đối xứng:** WebRTC channel có thể đạt end-to-end dưới 200ms. PSTN channel bắt đầu với 80-200ms chỉ cho truyền tải, chưa tính AI processing — latency budget bị ăn mất trước khi AI bắt đầu làm việc.

---

## Preprocessing pipeline — hai đường đến cùng AI Core

### Browser path (mỏng)

```
Microphone (48 kHz PCM)
    ↓ [Browser AEC + Noise Suppression — tự động]
    ↓ Opus Encode (adaptive bitrate)
    ↓ WebSocket → Server
    ↓ [Opus Decode]
    ↓ Downsample 48 kHz → 16 kHz
    → AI Model (PCM 16 kHz in)
```

Minimal processing. Opus 48kHz downsample xuống 16kHz — bỏ bớt thông tin, nhưng quality vẫn tốt vì bắt đầu từ 48kHz.

### Phone path (dày)

```
Twilio WebSocket (G.711 µ-law 8 kHz, base64 JSON)
    ↓ [Base64 decode → G.711 decode → PCM 8 kHz]
    ↓ Upsample 8 kHz → 16 kHz (nhân tạo — không phục hồi high freq bị cắt)
    ↓ [VAD, Barge-in logic, Echo suppress]
    → AI Model (PCM 16 kHz in)
    ↓ TTS output (24 kHz)
    ↓ Downsample 24 kHz → 8 kHz (bắt buộc với PSTN)
    ↓ G.711 Encode → base64 JSON
    → Twilio WebSocket → Phone
```

Heavy processing. Upsample 8→16kHz chỉ "giãn" narrowband signal — không thể phục hồi thông tin đã bị G.711 loại bỏ. Mỗi codec conversion là cơ hội tăng latency hoặc giảm chất lượng.

> **Preview Bài 04 (Audio Bridge):** Triển khai đúng hai pipeline này là challenge lớn nhất của Tầng 2. Chi tiết về codec chaining, quality loss tích lũy, và unified bridge pattern sẽ được phân tích ở Bài 04.

---

## Điểm hội tụ — "Channel chỉ là adapter, Core mới là bộ não"

Dù hai channel khác nhau hoàn toàn, chúng **hội tụ tại AI Core**:

- Cả hai đều đưa audio về **PCM 16kHz** trước khi vào AI model
- Cả hai chia sẻ chung **session state** (conversation history, context, tool results)
- Cả hai dùng chung **template/persona system**
- Cả hai qua cùng **post-call pipeline** (analytics, transcript, PII masking)

**Kiến trúc đúng:** AI Core independent khỏi channel — nhận PCM 16kHz, trả PCM 24kHz, không biết và không cần biết audio đến từ browser hay phone. Mỗi channel có **adapter layer riêng**:

- **Browser adapter:** Mỏng — WebSocket auth, Opus decode, downsample 48→16. Gần như pass-through.
- **Phone adapter:** Dày — JSON decode, G.711 decode, upsample 8→16, VAD, barge-in logic, echo suppress, downsample 24→8, G.711 encode.

```
Browser (Opus 48kHz)  →  Adapter mỏng  ─┐
                                         ├→  AI Core (PCM 16kHz in / 24kHz out)
Phone/PSTN (G.711 8kHz)  →  Adapter dày ─┘

AI Core không cần biết audio đến từ đâu.
```

Đây là ví dụ điển hình của **separation of concerns** trong system architecture — và là lý do scale model của hai channel hoàn toàn khác nhau:

- Browser adapter scale theo **số signaling connections** — nhẹ, không tốn compute
- Phone adapter scale theo **CPU (transcode)** — nặng, cần sticky session vì audio state nằm trong memory

---

## Tóm tắt — những gì cần nhớ

**PSTN và WebRTC là hai thế giới hoàn toàn khác nhau:**
PSTN: G.711 8kHz cố định, latency cao, 99.999% uptime, audio bắt buộc qua server. WebRTC: Opus 48kHz adaptive, latency thấp, phụ thuộc internet, audio có thể đi thẳng đến AI.

**WebRTC là stack hoàn chỉnh, không chỉ là giao thức:**
AEC, AGC, noise suppression, codec negotiation, NAT traversal — tất cả tích hợp miễn phí. PSTN không có những thứ này, server phải tự xử lý.

**Twilio/Telnyx là cầu nối bắt buộc với PSTN:**
Server không kết nối trực tiếp với mạng điện thoại. CPaaS nhận cuộc gọi PSTN và stream G.711 qua WebSocket đến server của bạn, với mỗi 20ms audio là một JSON message.

**"Channel chỉ là adapter, Core mới là bộ não":**
Thiết kế AI Core independent — nhận PCM 16kHz, trả PCM 24kHz, không quan tâm audio đến từ đâu. Browser adapter mỏng, Phone adapter dày. Đây là nguyên tắc kiến trúc quan trọng nhất từ bài này.

---

## Bài tiếp theo

**Bài 03 trả lời:** "Audio đến từ đâu?" → **WebRTC (browser) và PSTN (phone)**

**Bài 04 trả lời:** "Làm thế nào để xử lý hai nguồn audio này trong production?"

### Bài 04 — Audio Bridge — Demo vs Production

Có hai pipeline khác nhau trên giấy là một chuyện. Triển khai chúng trong production mà không tăng latency hay mất chất lượng là chuyện hoàn toàn khác. Bài 04 sẽ phân tích:

- Codec chaining và quality loss tích lũy qua nhiều bước
- Unified Audio Bridge cho cả hai channel trong một codebase
- Tại sao demo nghe tốt nhưng production nghe méo tiếng
- Ring buffer, backpressure, và memory management

→ [Đọc Bài 04 ngay](./04-audio-bridge.md)

---

*Bài 3/15 · ← [WebSocket & Realtime](./02-websocket-realtime-communication.md) · [Roadmap](./README.md) → [Audio Bridge](./04-audio-bridge.md)*
