# Bài 02 — WebSocket & Realtime Communication

> **Tầng 1 — Nền tảng** | Series: [Voice AI System Architecture](./README.md)
> ⏱ ~12 phút đọc · 💡 Sau bài này: hiểu tại sao Voice AI không thể dùng HTTP thông thường, và cách WebSocket hỗ trợ hai-chiều realtime streaming

---

## 🚨 Từ Demo Thành Production — Một Câu Chuyện Thực

Năm 2024, một startup AI healthcare demo được điều hành trên môi trường nội bộ. **Cuộc demo hoàn hảo:** Voice receptionist trả lời trong 280ms, giọng tự nhiên, context-aware. Investors thích thứ.

Ba tuần sau, startup **deploy lên production qua Twilio** (kết nối điện thoại thực tế).

- **Cuộc gọi #1-10:** Hoàn hảo.
- **Cuộc gọi #11:** Bị hang 2 giây, user chịu không được, cúp máy.
- **Cuộc gọi #50:** Timeout, "All agents busy" error.
- **Cuộc gọi #100+:** Dropout rate 40%, support tickets tăng vọt.

**Người ta đổ lỗi cho model:** "GPT-4o không đủ tốt," "Prompt engineering sai," "Cần fine-tune."

**Sai.** Model là 20% của problem. 80% là kiến trúc — mặc cụ thể: **cách truyền audio giữa client và server**.

Demo dùng HTTP POST requests. Production không chịu được. **Đây chính là lý do bạn cần học WebSocket — không phải optional, mà bắt buộc.**

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

#### ⚠️ **PRODUCTION GOTCHA: Ping/Pong Implementation Hell**

Lý thuyết vs thực tế là hai thứ khác nhau. Đây là bugs mà 90% engineers gặp lần đầu triển khai:

**Gotcha #1: Library không auto-reply ping**
Nhiều WebSocket library (đặc biệt JavaScript, Python) **không tự động reply pong** khi nhận ping. Bạn phải implement handler:
```javascript
ws.on('ping', () => ws.pong());  // Phải làm thủ công!
```
Quên cái này → Server timeout sau 30 giây → Call drop. Bạn sẽ spend 3 giờ debugging trước khi nhận ra.

**Gotcha #2: Timeout không đồng bộ**
Client nói "ping timeout sau 30s," server nói "ping timeout sau 15s." Kết quả: Một bên đợi, bên kia đã close connection. Luôn config cùng timeout trên server + client.

**Gotcha #3: Ping/pong qua các network gateway**
Nếu audio đi qua corporate firewall hay load balancer, ping/pong frame có thể bị drop. Firewall không hiểu WebSocket, coi ping là "vô dụng" và xóa. Result: Connection sống mà không gửi ping/pong → after 30 seconds, server thinks client dead nhưng client còn sống → desynchronization.

**Best Practice:**
- Explicit ping timeout: 30 seconds ✅
- Explicit pong timeout: 10 seconds (phải reply trong 10s, otherwise close) ✅
- Log mỗi ping/pong để debug: `[PING-OUT] [PONG-IN-OK] [PING-OUT]...` ✅
- Graceful close nếu pong timeout, không bắn error ✅

---

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
Tại sao gom HTTP + 2 WebSocket vào cùng process lại quan trọng — nếu dùng Redis:
- **Redis roundtrip per chunk:** 2-4ms (1-2ms write, 1-2ms read)
- **50 chunks/second:** 50 × 2-4ms = **100-200ms/second overhead**
- **Latency budget 300ms total:** 100-200ms là **33-66% của budget**
- **Còn lại cho AI:** Chỉ 100-150ms, không đủ để STT (30ms) + LLM (100ms) + TTS (100ms)

**Giải pháp:** Unified Server Pattern gom vào 1 process, dùng ring buffer (shared memory). Redis call = 0, shared memory call = <0.1ms. Latency < 50ms/second overhead thay vì 100-200ms.

**Đây là architectural catastrophe.** Nếu bạn thiết kế distributed system với Redis sync cho Voice AI, bạn sẽ tốn 6 tháng debug tại sao "AI response chậm quá," không biết là protocol overhead chứ không phải model.

**3. Graceful Shutdown**
Tại sao graceful shutdown quan trọng — server restart = đóng WebSocket = rơi call. Cần timeout đủ lâu để kết thúc cleanly.

**4. Latency Budget**
Tại sao overhead thấp của WebSocket so với HTTP là yếu tố sống còn — mỗi millisecond đều quan trọng.

**5. Kiến trúc hai channel**
Tại sao cùng một AI Core nhưng Browser channel (Direct WebSocket) và Phone channel (Double WebSocket) có kiến trúc hoàn toàn khác nhau.

<style>
  .wrap{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif;padding:.5rem 0}
  .section{margin:0 0 2rem}
  .section-title{font-size:14px;font-weight:600;color:#1f2937;margin:0 0 10px}
  .vs{display:grid;grid-template-columns:1fr 1fr;gap:10px;margin:0 0 12px}
  .vs-card{border-radius:8px;padding:14px 16px;border:.5px solid #e5e7eb;background:#ffffff}
  .vs-h{font-size:13px;font-weight:600;margin:0 0 10px;display:flex;align-items:center;gap:6px;color:#1f2937}
  .vs-dot{width:8px;height:8px;border-radius:50%;flex-shrink:0}
  .vs-row{display:flex;justify-content:space-between;font-size:12px;padding:5px 0;border-top:.5px solid #e5e7eb}
  .vs-row:first-of-type{border-top:none}
  .vs-k{color:#6b7280}
  .vs-v{color:#1f2937;font-weight:600;text-align:right;max-width:55%}
  .timeline{margin:0 0 8px}
  .tl-label{font-size:11px;color:#6b7280;margin:0 0 6px;letter-spacing:.03em;text-transform:uppercase}
  .tl-row{display:flex;align-items:center;gap:0;margin:0 0 4px}
  .tl-box{font-size:10px;padding:5px 8px;border-radius:6px;text-align:center;line-height:1.3;font-weight:600}
  .tl-arr{font-size:12px;color:#9ca3af;padding:0 2px;flex-shrink:0}
  .tl-http .tl-box{background:#fee2e2;color:#991b1b}
  .tl-ws .tl-box{background:#d1fae5;color:#065f46}
  .note-box{border-radius:6px;padding:10px 14px;font-size:12px;line-height:1.5;margin:10px 0 0}
  .note-info{background:#eff6ff;color:#1e40af;border-left:3px solid #3b82f6}
  .note-warn{background:#fef3c7;color:#92400e;border-left:3px solid #fbbf24}
  .pattern{background:#f3f4f6;border-radius:8px;padding:14px 16px;margin:0 0 10px}
  .pattern-h{font-size:13px;font-weight:600;color:#1f2937;margin:0 0 8px}
  .pattern-flow{display:flex;align-items:center;gap:0;flex-wrap:wrap;margin:0 0 6px}
  .pattern-node{font-size:11px;padding:6px 10px;border-radius:6px;text-align:center;line-height:1.3;white-space:nowrap;font-weight:500}
  .n-blue{background:#dbeafe;color:#1e40af}
  .n-green{background:#d1fae5;color:#065f46}
  .n-purple{background:#e9d5ff;color:#6b21a8}
  .n-gray{background:#f3f4f6;color:#4b5563;border:.5px solid #e5e7eb}
  .p-arr{font-size:12px;color:#9ca3af;padding:0 3px;flex-shrink:0}
  .pattern-note{font-size:11px;color:#4b5563;line-height:1.4;margin:0}
  @media(max-width:480px){.vs{grid-template-columns:1fr}}
</style>
<div class="wrap">

<div class="section">
  <div class="section-title">HTTP vs WebSocket cho audio streaming</div>
  <div class="vs">
    <div class="vs-card">
      <div class="vs-h"><div class="vs-dot" style="background:#dc2626"></div> HTTP request-response</div>
      <div class="vs-row"><span class="vs-k">Mô hình</span><span class="vs-v">Hỏi → đáp → đóng</span></div>
      <div class="vs-row"><span class="vs-k">Hướng</span><span class="vs-v">Một chiều mỗi lượt</span></div>
      <div class="vs-row"><span class="vs-k">Header overhead</span><span class="vs-v">200-500 bytes/request</span></div>
      <div class="vs-row"><span class="vs-k">Connection</span><span class="vs-v">Mở/đóng mỗi lần</span></div>
      <div class="vs-row"><span class="vs-k">Server push?</span><span class="vs-v">Không thể</span></div>
      <div class="vs-row"><span class="vs-k">Audio 20ms chunks</span><span class="vs-v">50 req/s mỗi chiều</span></div>
    </div>
    <div class="vs-card">
      <div class="vs-h"><div class="vs-dot" style="background:#059669"></div> WebSocket (RFC 6455)</div>
      <div class="vs-row"><span class="vs-k">Mô hình</span><span class="vs-v">Mở 1 lần → stream liên tục</span></div>
      <div class="vs-row"><span class="vs-k">Hướng</span><span class="vs-v">Full-duplex (cả 2 chiều)</span></div>
      <div class="vs-row"><span class="vs-k">Frame overhead</span><span class="vs-v">2-14 bytes/frame</span></div>
      <div class="vs-row"><span class="vs-k">Connection</span><span class="vs-v">Persistent suốt cuộc gọi</span></div>
      <div class="vs-row"><span class="vs-k">Server push?</span><span class="vs-v">Bất cứ lúc nào</span></div>
      <div class="vs-row"><span class="vs-k">Audio 20ms chunks</span><span class="vs-v">Frames liên tục, gần 0 overhead</span></div>
    </div>
  </div>
</div>

<div class="section">
  <div class="section-title">Gửi 1 giây audio (50 chunks × 20ms)</div>
  <div class="timeline">
    <div class="tl-label">HTTP — mỗi chunk là 1 request</div>
    <div class="tl-row tl-http">
      <div class="tl-box">Req 1<br>+header</div><span class="tl-arr">→</span>
      <div class="tl-box">Res 1</div><span class="tl-arr">,</span>
      <div class="tl-box">Req 2<br>+header</div><span class="tl-arr">→</span>
      <div class="tl-box">Res 2</div><span class="tl-arr">,</span>
      <div class="tl-box" style="opacity:.6">...×50</div>
    </div>
  </div>
  <div class="timeline">
    <div class="tl-label">WebSocket — 1 connection, frames liên tục</div>
    <div class="tl-row tl-ws">
      <div class="tl-box">Handshake<br>(1 lần)</div><span class="tl-arr">→</span>
      <div class="tl-box">Frame 1</div><span class="tl-arr">→</span>
      <div class="tl-box">Frame 2</div><span class="tl-arr">→</span>
      <div class="tl-box">Frame 3</div><span class="tl-arr">→</span>
      <div class="tl-box" style="opacity:.7">...liên tục</div>
    </div>
  </div>
  <div class="note-box note-warn">Audio G.711: mỗi chunk 20ms = ~320 bytes. HTTP header thêm ~300 bytes → overhead gần 50%. WebSocket frame header chỉ 2 bytes → overhead dưới 1%.</div>
</div>

<div class="section">
  <div class="section-title">2 pattern WebSocket trong Voice AI</div>
  <div class="pattern">
    <div class="pattern-h">Pattern 1: Browser — Direct WebSocket</div>
    <div class="pattern-flow">
      <span class="pattern-node n-blue">Browser</span>
      <span class="p-arr">—token→</span>
      <span class="pattern-node n-gray">Server (auth only)</span>
      <span class="p-arr">—token→</span>
      <span class="pattern-node n-blue">Browser</span>
      <span class="p-arr">══WSS══</span>
      <span class="pattern-node n-green">AI Model</span>
    </div>
    <p class="pattern-note">Server không chạm audio → latency tối thiểu. Audio đi thẳng browser ↔ AI.</p>
  </div>
  <div class="pattern">
    <div class="pattern-h">Pattern 2: Phone — Double WebSocket qua Server</div>
    <div class="pattern-flow">
      <span class="pattern-node n-purple">Phone/PSTN</span>
      <span class="p-arr">→</span>
      <span class="pattern-node n-purple">Twilio</span>
      <span class="p-arr">═WS#1═</span>
      <span class="pattern-node n-gray">Server (transcode)</span>
      <span class="p-arr">═WS#2═</span>
      <span class="pattern-node n-green">AI Model</span>
    </div>
    <p class="pattern-note">Server xử lý transcode G.711↔PCM. 2 WebSocket + audio buffer + session state nằm cùng process.</p>
  </div>
  <div class="note-box note-info">Đây là lý do Unified Server pattern (chủ đề 12) gom HTTP + 2 WebSocket vào 1 process — sync qua Redis mỗi 20ms sẽ giết latency.</div>
</div>

</div>

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

## Bài tiếp theo — Chuỗi Kiến Thức

**Bài 02 trả lời:** "Truyền dữ liệu **thế nào**?" → **WebSocket** (two-way realtime)

**Bài 03 trả lời:** "Truyền **cái gì**?" → **Audio Codec** (Opus 48kHz vs G.711 8kHz)

---

### 🔗 **Bài 03 — Browser vs Phone — 2 Thế Giới Kết Nối**

**Tưởng tượng:** Cùng một WebSocket connection, nhưng:
- **Browser gửi:** Opus 48 kHz (full bandwidth, natural sound) ✨
- **Phone gửi:** G.711 8 kHz (narrowband, tinny sound) 📞

**Cùng một AI Model,** nhưng server cần hai "adapter" hoàn toàn khác nhau để xử lý.

**Và đây là lý do 80% production bugs không phải từ model,** mà từ **codec mismatch, resampling error, echo cancellation fail.**

Bài 02 bạn vừa học: Protocol (WebSocket) ✅  
Bài 03 sẽ dạy: Audio format (codec) ← đây là nơi tiền game thật bắt đầu

→ [Đọc Bài 03 ngay](./03-browser-vs-phone.md)

---

*Bài 2/15 · ← [Audio Fundamentals](./01-audio-fundamentals.md) · [Roadmap](./README.md) → [Browser vs Phone](./03-browser-vs-phone.md)*
