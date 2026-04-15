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

### Chi tiết sâu: Unified Server Pattern vs Distributed Architecture

**Tình huống điển hình:** Nhiều engineer khi nhìn Pattern 2 (Phone Double WebSocket), sẽ tự nhiên nghĩ:

> *"Tại sao không tách thành 2 service riêng? Một service nhận WS#1 từ Twilio, một service kết nối WS#2 đến AI Model, rồi sync qua Redis?"*

Ý tưởng này hợp lý trên giấy — microservices, scale independently, decoupled systems. Nhưng **thực tế trong Voice AI, nó sẽ giết latency**.

#### Tại sao Redis sync giết latency?

Hãy tính toán chi tiết:

**Redis roundtrip latency:**
- Local Redis (same data center): ~1-2ms per operation
- Network Redis (different region): ~5-10ms per operation

**Per 20ms audio chunk:**
- Mỗi chunk: 1 write (WS#1 → Redis) + 1 read (Redis → WS#2) = **2-4ms latency**
- Còn lại: 16-18ms cho xử lý thực tế (transcode, VAD, AI processing)

**50 chunks mỗi giây:**
- 50 × 2-4ms = **100-200ms/giây overhead**

**Latency budget 300ms total:**
- Protocol overhead (WebSocket handshake, frame headers): ~20-50ms
- Redis sync overhead: 100-200ms
- **Total overhead: 120-250ms — đó là 40-83% của budget**
- Còn cho AI: chỉ 50-180ms

**Result:** AI bị "cắt cơm" — không có thời gian suy luận, chỉ kịp nhận input, xử lý tối thiểu, trả output.

#### 4 vấn đề chính khi dùng Redis distributed architecture

**1. Redis Roundtrip Latency**
- Mỗi write: application → Redis network → Redis server → store
- Mỗi read: application → Redis network → Redis server → fetch
- Worst case: 10ms write + 10ms read = 20ms một chunk
- Best case: 1ms write + 1ms read = 2ms một chunk
- Khi bạn làm 50 times/second, cộng lại = hàng trăm ms mất đi

**2. 50 Chunks Per Second**
- Voice AI audio stream: 20ms packets = 50 packets/second
- Mỗi packet đi qua Redis = latency
- Scale vấn đề: "Chỉ" 2-4ms per chunk × 50 = **100-200ms toàn bộ**

**3. Latency Budget Impact**
- Total budget: 300ms
- Redis overhead: 100-200ms
- **Phần trăm tổn thất: 33-66%**
- Con số này không chấp nhận được cho conversational AI

**4. AI Processing Time Left**
- Sau Redis sync, còn ~100-150ms
- STT: 20-30ms
- LLM inference: 50-100ms
- TTS generation: 50-100ms
- **Kết quả: Model không đủ thời gian suy luận, câu trả lời bị "vội vàng", quality tệ**

#### Giải pháp: Unified Server Pattern

**Triết lý:** Gom HTTP + 2 WebSocket vào **1 Node.js process duy nhất**, giao tiếp qua **shared memory** (không Redis).

**Architecture:**
```
Twilio WS#1 ──→ Server Process ──→ AI Model WS#2
                ↓
         Ring Buffer (Shared Memory)
         ↓
    Processing Thread
    (Transcode, VAD, Echo suppress)
```

**Chi tiết:**
1. **WS#1 receives** G.711 8kHz from Twilio → write directly to ring buffer (shared memory)
2. **Processing thread** reads from ring buffer → transcode to 16kHz → send to AI Model WS#2
3. **Ring buffer** = fixed-size circular queue in process memory
4. **All operations** in same process = **no network round-trip**

**Latency per chunk:**
- Ring buffer write: **<0.1ms** (memory operation, not network)
- Ring buffer read: **<0.1ms**
- Transcode: **1-2ms** (local CPU)
- **Total: <1ms per chunk**

**Result:**
- 50 chunks × <1ms = **<50ms toàn bộ overhead**
- Còn cho AI: **~250ms cho suy luận đầy đủ**
- AI có đủ thời gian cho high-quality response

#### Chi tiết Implementation

**Ring Buffer Pattern:**
```
┌─────────────────────────────────────┐
│     Shared Memory Ring Buffer        │
├─────────────────────────────────────┤
│  [chunk 1] [chunk 2] [chunk 3] ...  │
│    ↑                        ↑        │
│  write_ptr (Twilio WS#1)  read_ptr  │
│                        (Processing) │
└─────────────────────────────────────┘
```

**Size:** Typically 100-500ms worth of audio (2-10 chunks overlap)

**Thread Safety:** Lock-free queue (atomic operations) or mutex (simpler, sufficient for <1ms operations)

**Backpressure:** If read_ptr catches up to write_ptr, pause Twilio receiving (briefly buffer in Twilio SDK).

#### Tại sao Unified Server Pattern không scale-out?

"Nhưng nếu server lớn, tôi muốn chia cho nhiều instance?"

**Câu trả lời:** Không thể — audio stream từ single phone call phải ở 1 process để share ring buffer. Bạn không thể split một phone call qua 2 servers mà không giới thiệu Redis (rơi vào trap ở trên).

**Scaling strategy khác:**
- 1 call = 1 Unified Server instance
- Horizontal scale = 1000 concurrent calls = 1000 instances
- Load balancer route: phone number → consistent instance
- Không có shared state giữa instances (state isolated per call)

#### Đây là điểm mấu chốt

> Hiểu WebSocket không chỉ là hiểu protocol, mà là hiểu **tại sao kiến trúc server phải là sao** để tránh latency trap.

Pattern 2 (Double WebSocket) chỉ hoạt động tốt khi triển khai qua Unified Server Pattern. Nếu cố gắng distributed + Redis, bạn sẽ:
- Mất 33-66% latency budget
- AI không có thời gian xử lý → quality tệ
- Người dùng cảm nhận độ trễ không tự nhiên (>600ms feel broken)

Bài 12 sẽ đi sâu vào Unified Server Pattern implementation details. Nhưng nền tảng hiểu **tại sao** nó cần thiết phải bắt đầu từ đây — từ việc hiểu WebSocket và latency trade-offs.

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
