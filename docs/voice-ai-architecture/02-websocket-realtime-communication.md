# Bài 02 — WebSocket & Realtime Communication

> **Tầng 1 — Nền tảng** | Series: [Voice AI System Architecture](./README.md)
> ⏱ ~12 phút đọc · 💡 Sau bài này: hiểu tại sao Voice AI không thể dùng HTTP thông thường, và cách WebSocket hỗ trợ hai-chiều realtime streaming

---

## Bài toán cốt lõi — Tại sao Voice AI không thể dùng HTTP?

**HTTP hoạt động theo mô hình request-response:** Client gửi request → Server xử lý → Server trả response → Kết nối đóng.

Mỗi lần muốn dữ liệu mới, client phải gửi request mới. Giống như gửi thư — mỗi lần muốn hỏi gì đó, bạn phải viết thư mới, gửi đi, chờ hồi âm.

### Vấn đề với Voice AI

Cuộc hội thoại voice là **liên tục và hai chiều đồng thời**. Khi bạn nói chuyện với AI qua điện thoại:

- Cả hai bên đều có thể "nói" bất cứ lúc nào
- Audio là dòng dữ liệu liên tục — cứ mỗi 20ms lại có chunk audio mới
- Cần truyền **50 chunks mỗi giây, hai chiều cùng lúc**

Với HTTP, mỗi chunk cần một request riêng kèm:
- Header overhead: ~200-500 bytes (so với payload 320 bytes)
- Mở/đóng connection: thêm TCP + TLS handshake
- Request/response cycle: phải chờ response trước khi gửi cái tiếp theo

**Kết quả:** Overhead có thể lên đến 50-150% với HTTP, trong khi latency budget Voice AI chỉ có 300ms. Toàn bộ budget sẽ bị lấy mất bởi protocol overhead.

---

## Những nỗ lực "hack" HTTP để realtime

Trước khi WebSocket ra đời (2011), developer đã cố gắng tạo realtime trên HTTP bằng các kỹ thuật:

### Polling
Client gửi request liên tục theo chu kỳ (ví dụ mỗi 100ms) để hỏi "có dữ liệu mới không?". 

**Vấn đề:** Đa số request trả về rỗng, tốn băng thông. Không đủ nhanh cho audio realtime.

### Long Polling
Client gửi request, server giữ connection mở cho đến khi có dữ liệu mới rồi mới trả response.

**Vấn đề:** Vẫn một chiều. Có khoảng "chết" giữa response khi client phải gửi request mới.

### Server-Sent Events (SSE)
Server push dữ liệu liên tục xuống client qua HTTP connection mở.

**Vấn đề:** Chỉ một chiều (server → client). Client gửi ngược lại phải mở request riêng. Không phù hợp cho audio hai chiều.

---

## WebSocket — Giải pháp được thiết kế cho realtime

**WebSocket (RFC 6455, 2011)** là giao thức truyền thông **hai chiều, full-duplex, persistent** trên một TCP connection duy nhất.

Cả client và server đều có thể gửi dữ liệu bất cứ lúc nào mà không cần request/response cycle.

### Cách WebSocket hoạt động

1. **Opening Handshake** — HTTP Upgrade request với header `Upgrade: websocket`
2. **Server đồng ý** — Response 101 (Switching Protocols)
3. **Kết nối thiết lập** — Từ đây, connection TCP không còn nói HTTP
4. **Two-way streaming** — Cả hai bên gửi nhận frames liên tục suốt cuộc gọi

### Frame Structure

Mỗi WebSocket frame có header tối thiểu **2-14 bytes** (so với hàng trăm bytes của HTTP header).

Ví dụ: Audio chunk 20ms = 320 bytes (G.711 8kHz)
- **HTTP:** ~300 bytes header + 320 bytes payload = 50% overhead
- **WebSocket:** ~2 bytes header + 320 bytes payload = < 1% overhead

---

## Đặc điểm WebSocket quan trọng cho Voice AI

### 1. Full-duplex (Hai chiều đồng thời)

Cả hai bên gửi nhận trên **cùng một connection**, không phải chờ lượt.

Khi user đang nói (gửi audio lên), AI cũng có thể đang nói (gửi audio xuống). Đây là nền tảng cho **barge-in** — user nói chen ngang.

### 2. Overhead cực thấp

Sau handshake ban đầu, mỗi frame chỉ tốn 2-14 bytes header.

Với audio chunk 20ms, overhead chiếm dưới 5% — so với HTTP có thể 60-150%.

### 3. Persistent Connection

Một connection TCP duy nhất được tái sử dụng suốt cuộc gọi.

Tiết kiệm TCP handshake (1 RTT) và TLS handshake (1-2 RTT) mỗi lần.

### 4. Latency tối thiểu

Không có request/response cycle. Dữ liệu gửi gần như tức thì.

Latency phụ thuộc chủ yếu vào network round-trip, không phải protocol overhead.

---

## WebSocket trong kiến trúc Voice AI — Hai pattern khác nhau

### Pattern 1: Browser Direct WebSocket

```
Browser ──token──→ Server (auth only)
         ◀─token─
         
Browser ═══WSS═══→ AI Model (Direct)
         ◀────────
         (Audio)
```

**Đặc điểm:**
- Server chỉ làm **authentication** (cấp ephemeral token)
- Audio đi **thẳng browser ↔ AI**, server không chạm
- Latency tối thiểu vì audio không qua server

**Ứng dụng:** Web app, mobile app qua browser

### Pattern 2: Phone Double WebSocket

```
Phone/PSTN ──→ Twilio/Telnyx
              ═WSS#1═
                    Server (transcode)
                    ═WSS#2═
                           AI Model
```

**Đặc điểm:**
- **WSS #1:** Twilio/Telnyx stream G.711 (8kHz) → Server
- **WSS #2:** Server stream PCM 16kHz → AI Model
- Server ở giữa **transcode** audio: G.711 → PCM → upsample → forward
- Server cũng quản lý **session state** (cuộc gọi ID, transcript, v.v.)

**Ứng dụng:** Điện thoại thực (PSTN), gọi qua số điện thoại

---

## Connection lifecycle — Từ mở đến đóng

### Giai đoạn 1: Opening Handshake

HTTP Upgrade request → 101 response. Mất **1 RTT**.

Nếu dùng WSS (secure), thêm TLS handshake. Tổng cộng **50-150ms** cho connection cùng region.

### Giai đoạn 2: Active Communication

Frames truyền hai chiều liên tục. Kéo dài suốt cuộc gọi (có thể hàng phút).

Cả hai bên gửi:
- **Data frames:** Audio chunks
- **Ping/pong frames:** Keep-alive signal

### Giai đoạn 3: Ping/Pong (Heartbeat)

WebSocket có cơ chế ping/pong tích hợp. Server gửi ping frame định kỳ (15-30 giây), client phải trả pong.

**Quan trọng cho Voice AI:** Nếu không nhận pong, server biết client đã disconnect → cleanup resources (audio buffer, session state).

Nếu không phát hiện, resources sẽ leak.

### Giai đoạn 4: Closing Handshake

Một bên gửi close frame, bên kia acknowledge. Connection TCP đóng gracefully.

**Status code cho biết lý do:**
- `1000` — Normal closure
- `1001` — Going away (user tắt browser)
- `1006` — Abnormal closure (mất kết nối đột ngột)

Server dùng thông tin này để quyết định post-call handling:
- Kết thúc bình thường → trigger analytics
- Disconnect bất thường → log error, có thể retry

---

## WSS — WebSocket Secure

**WSS (WebSocket Secure)** = WebSocket trên TLS, tương tự HTTPS với HTTP.

### Bắt buộc trong production vì:

**1. Bảo mật dữ liệu**
Audio là dữ liệu nhạy cảm (thông tin cá nhân, y tế, tài chính). TLS mã hóa toàn bộ.

**2. Browser requirement**
Hầu hết browser modern block WebSocket không mã hóa (ws://) từ trang HTTPS. Voice AI app chạy HTTPS → cần WSS.

**3. Twilio/Telnyx requirement**
Nhà cung cấp telephony chỉ stream audio qua WSS, không chấp nhận WS.

### Overhead

TLS handshake lúc kết nối: **50-100ms thêm** (một lần duy nhất).

Mã hóa/giải mã mỗi frame: vài **microsecond** — không đáng kể so với latency budget 300ms.

---

## So sánh WebSocket với giao thức realtime khác

### WebRTC (Web Real-Time Communication)

Giao thức **peer-to-peer** cho audio/video realtime trên browser.

**Khác WebSocket:**
- WebRTC tối ưu cho media (codec negotiation, echo cancellation, bandwidth adaptation)
- WebSocket là giao thức truyền dữ liệu tổng quát

**Trong Voice AI:** Browser channel dùng WebRTC cho audio, WebSocket cho signaling/control.

### gRPC Streaming

Framework RPC hỗ trợ bidirectional streaming qua HTTP/2.

**Ưu điểm:** Typed contracts (protobuf), built-in load balancing.

**Nhược điểm:** Phức tạp, không native trên browser (cần gRPC-web proxy).

### MQTT

Giao thức pub/sub nhẹ, phổ biến trong IoT.

**Nhược điểm:** Thiết kế cho message passing, không tối ưu cho continuous streaming.

---

## Tại sao hiểu WebSocket quan trọng

Nếu không hiểu WebSocket, bạn sẽ không hiểu:

**1. Double WebSocket Pattern**
Tại sao phone channel cần 2 WebSocket riêng (Twilio → Server, Server → AI) và tại sao chúng phải giao tiếp qua shared memory của 1 process.

**2. Unified Server (Chủ đề 12)**
Tại sao gom HTTP + 2 WebSocket vào cùng process lại quan trọng — sync qua Redis mỗi 20ms sẽ giết latency.

**3. Graceful Shutdown**
Tại sao graceful shutdown quan trọng — server restart = đóng WebSocket = rơi call. Cần timeout đủ lâu để kết thúc cleanly.

**4. Latency Budget**
Tại sao overhead thấp của WebSocket so với HTTP là yếu tố sống còn — mỗi millisecond đều quan trọng.

**5. Kiến trúc hai channel**
Tại sao cùng một AI Core nhưng Browser channel (Direct WebSocket) và Phone channel (Double WebSocket) có kiến trúc hoàn toàn khác nhau.

---

## Tóm tắt — Những gì cần nhớ

**HTTP không phù hợp cho Voice AI** vì mô hình request-response tạo overhead quá lớn khi cần truyền 50+ chunks/giây, hai chiều đồng thời. Server cũng không thể push dữ liệu chủ động.

**WebSocket là persistent, full-duplex, low-overhead.** Một connection mở suốt cuộc gọi, cả hai bên gửi nhận đồng thời, frame header chỉ 2-14 bytes.

**Hai pattern khác nhau tùy channel:**
- Browser: Direct WebSocket (audio thẳng browser ↔ AI)
- Phone: Double WebSocket qua server (server transcode audio)

**Connection lifecycle quan trọng.** Ping/pong phát hiện disconnect, close frame cho biết lý do, graceful shutdown tránh rơi call.

**WSS bắt buộc production** vì browser, Twilio/Telnyx đều yêu cầu — overhead của TLS không đáng kể.

---

## Bài tiếp theo

Bài 03 — **Browser vs Phone — 2 Thế Giới Kết Nối** sẽ xây dựng trên nền tảng WebSocket này.

Sẽ giải thích chi tiết:
- WebRTC cho browser (cấu trúc codec, streaming quality)
- PSTN + Twilio cho phone (codec G.711, limitations)
- Tại sao chúng tạo ra hai "thế giới" hoàn toàn khác nhau về audio quality, latency, kiến trúc

---

*Bài 2/15 · ← [Audio Fundamentals](./01-audio-fundamentals.md) · [Roadmap](./README.md) → [Browser vs Phone](./03-browser-vs-phone.md)*
