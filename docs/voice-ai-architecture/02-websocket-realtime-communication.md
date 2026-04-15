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

## P9 — HTTP vs WebSocket Overhead: Visual Comparison

<div class="html-component">
<style>
.overhead-section { font-family: "Segoe UI", sans-serif; margin: 20px 0; }
.overhead-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; margin: 20px 0; }
.overhead-visual {
  padding: 20px;
  background: #f9f9f9;
  border-radius: 8px;
  border-left: 3px solid;
}
.overhead-visual.http { border-left-color: #dc2626; }
.overhead-visual.ws { border-left-color: #1D9E75; }
.overhead-title { font-weight: 600; font-size: 13px; margin-bottom: 10px; }
.overhead-stream {
  display: flex;
  flex-direction: column;
  gap: 4px;
  font-size: 10px;
}
.overhead-frame {
  display: flex;
  height: 24px;
  align-items: center;
  gap: 2px;
  background: white;
  border: 1px solid #e5e7eb;
  border-radius: 3px;
  padding: 2px 4px;
  overflow: hidden;
}
.overhead-header {
  background: #fee2e2;
  color: #991b1b;
  padding: 2px 4px;
  border-radius: 2px;
  font-size: 9px;
  font-weight: 600;
  white-space: nowrap;
}
.overhead-payload {
  background: #d1fae5;
  color: #065f46;
  padding: 2px 4px;
  border-radius: 2px;
  font-size: 9px;
  flex: 1;
  white-space: nowrap;
}
.overhead-wait {
  height: 20px;
  background: linear-gradient(90deg, #fee2e2 0%, #fee2e2 100%);
  border-radius: 3px;
  padding: 2px 4px;
  font-size: 9px;
  color: #991b1b;
  display: flex;
  align-items: center;
  font-weight: 600;
}
.overhead-stats {
  margin-top: 12px;
  padding: 8px;
  background: white;
  border-radius: 4px;
  font-size: 10px;
  border: 1px solid #e5e7eb;
}
.overhead-stat-row { display: flex; justify-content: space-between; margin: 3px 0; }
.overhead-stat-label { color: #666; }
.overhead-stat-value { font-weight: 600; }
.http .overhead-stat-value { color: #dc2626; }
.ws .overhead-stat-value { color: #1D9E75; }
@media (max-width: 640px) { .overhead-grid { grid-template-columns: 1fr; } }
</style>

<div class="overhead-section">
  <p style="font-size:12px; color:#666; margin-bottom:15px;">📊 Gửi 50 audio chunks trong 1 giây (mỗi chunk 20ms = 320 bytes)</p>

  <div class="overhead-grid">
    <div class="overhead-visual http">
      <div class="overhead-title">❌ HTTP — Mỗi chunk = 1 request riêng</div>
      <div class="overhead-stream">
        <div class="overhead-frame">
          <div class="overhead-header" style="width:40%;">Header<br>~300 bytes</div>
          <div class="overhead-payload" style="width:35%;">Payload<br>320 bytes</div>
        </div>
        <div class="overhead-wait">⏳ Wait for response...</div>
        <div class="overhead-frame">
          <div class="overhead-header" style="width:40%;">Header</div>
          <div class="overhead-payload" style="width:35%;">Payload</div>
        </div>
        <div class="overhead-wait">⏳ Wait for response...</div>
        <div class="overhead-frame" style="opacity:0.6;">
          <div class="overhead-header" style="width:40%;">...</div>
          <div class="overhead-payload" style="width:35%;">...</div>
        </div>
        <div style="text-align:center; font-size:10px; color:#666; margin-top:4px;">×50 requests</div>
      </div>
      <div class="overhead-stats">
        <div class="overhead-stat-row">
          <span class="overhead-stat-label">Total requests:</span>
          <span class="overhead-stat-value">50</span>
        </div>
        <div class="overhead-stat-row">
          <span class="overhead-stat-label">Header wasted per request:</span>
          <span class="overhead-stat-value">~300 bytes</span>
        </div>
        <div class="overhead-stat-row">
          <span class="overhead-stat-label">Total overhead:</span>
          <span class="overhead-stat-value">15KB wasted</span>
        </div>
        <div class="overhead-stat-row">
          <span class="overhead-stat-label">Overhead %:</span>
          <span class="overhead-stat-value">~50%</span>
        </div>
        <div class="overhead-stat-row" style="margin-top:6px; padding-top:6px; border-top:1px solid #e5e7eb;">
          <span class="overhead-stat-label" style="color:#666;"><strong>User experience:</strong></span>
          <span class="overhead-stat-value" style="color:#dc2626;">Choppy, gaps</span>
        </div>
      </div>
    </div>

    <div class="overhead-visual ws">
      <div class="overhead-title">✅ WebSocket — 50 frames streaming continuously</div>
      <div class="overhead-stream">
        <div class="overhead-frame">
          <div class="overhead-header" style="width:5%; background:#dbeafe; color:#1e40af;">2b</div>
          <div class="overhead-payload" style="width:95%; background:#dbeafe; color:#1e40af;">Frame 1 (320 bytes payload)</div>
        </div>
        <div class="overhead-frame">
          <div class="overhead-header" style="width:5%; background:#dbeafe; color:#1e40af;">2b</div>
          <div class="overhead-payload" style="width:95%; background:#dbeafe; color:#1e40af;">Frame 2 (320 bytes payload)</div>
        </div>
        <div class="overhead-frame">
          <div class="overhead-header" style="width:5%; background:#dbeafe; color:#1e40af;">2b</div>
          <div class="overhead-payload" style="width:95%; background:#dbeafe; color:#1e40af;">Frame 3 (320 bytes payload)</div>
        </div>
        <div style="text-align:center; font-size:10px; color:#666; margin-top:4px;">...liên tục, không wait</div>
      </div>
      <div class="overhead-stats">
        <div class="overhead-stat-row">
          <span class="overhead-stat-label">Total frames:</span>
          <span class="overhead-stat-value">50</span>
        </div>
        <div class="overhead-stat-row">
          <span class="overhead-stat-label">Header per frame:</span>
          <span class="overhead-stat-value">2-8 bytes</span>
        </div>
        <div class="overhead-stat-row">
          <span class="overhead-stat-label">Total overhead:</span>
          <span class="overhead-stat-value">~300 bytes</span>
        </div>
        <div class="overhead-stat-row">
          <span class="overhead-stat-label">Overhead %:</span>
          <span class="overhead-stat-value">&lt;1%</span>
        </div>
        <div class="overhead-stat-row" style="margin-top:6px; padding-top:6px; border-top:1px solid #d1fae5;">
          <span class="overhead-stat-label" style="color:#666;"><strong>User experience:</strong></span>
          <span class="overhead-stat-value" style="color:#1D9E75;">Smooth, real-time</span>
        </div>
      </div>
    </div>
  </div>

  <div style="background:#eff6ff; border-left:3px solid #3b82f6; padding:12px; border-radius:4px; margin-top:15px; font-size:11px;">
    <strong>🔑 Key insight:</strong> WebSocket overhead &lt;1%, HTTP overhead ~50%. That's why HTTP fails for Voice AI realtime — không phải vì protocol slow, mà vì overhead structure. Mỗi request-response cycle = delay.
  </div>
</div>

</div>

---

## WebSocket — Giải pháp được thiết kế cho realtime

**WebSocket (RFC 6455, 2011)** là giao thức truyền thông **hai chiều, full-duplex, persistent** trên một TCP connection duy nhất.

Cả client và server đều có thể gửi dữ liệu bất cứ lúc nào mà không cần request/response cycle.

### Cách WebSocket hoạt động

1. **Opening Handshake** — HTTP Upgrade request với header `Upgrade: websocket`
2. **Server đồng ý** — Response 101 (Switching Protocols)
3. **Kết nối thiết lập** — Từ đây, connection TCP không còn nói HTTP
4. **Two-way streaming** — Cả hai bên gửi nhận frames liên tục suốt cuộc gọi

### Frame Structure — Deep Dive

Mỗi WebSocket frame có header tối thiểu **2-14 bytes** (so với hàng trăm bytes của HTTP header).

<div class="html-component">
<style>
.frame-section { font-family: "Segoe UI", sans-serif; margin: 20px 0; }
.frame-scenarios { display: grid; gap: 15px; margin: 20px 0; }
.frame-scenario {
  border: 1px solid #e5e7eb;
  border-radius: 6px;
  padding: 15px;
  background: #f9f9f9;
}
.frame-scenario-title { font-weight: 600; font-size: 12px; margin-bottom: 10px; color: #1f2937; }
.frame-diagram {
  display: flex;
  gap: 4px;
  align-items: center;
  margin-bottom: 10px;
  flex-wrap: wrap;
  font-size: 10px;
}
.frame-byte {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 32px;
  min-width: 32px;
  border: 1px solid #d1d5db;
  border-radius: 3px;
  font-weight: 600;
  font-size: 10px;
  position: relative;
  background: white;
}
.frame-byte.b0 { background: #dbeafe; color: #1e40af; border-color: #93c5fd; }
.frame-byte.b1 { background: #fef3c7; color: #92400e; border-color: #fcd34d; }
.frame-byte.b2-3 { background: #d1fae5; color: #065f46; border-color: #a7f3d0; }
.frame-byte.b4-7 { background: #e9d5ff; color: #6b21a8; border-color: #d8b4fe; }
.frame-byte.payload { background: #fee2e2; color: #991b1b; border-color: #fca5a5; min-width: 60px; }
.frame-label-small { font-size: 8px; color: #666; margin-top: 2px; text-align: center; }
.frame-breakdown {
  display: grid;
  grid-template-columns: auto 1fr;
  gap: 12px;
  margin-top: 10px;
  padding-top: 10px;
  border-top: 1px solid #e5e7eb;
  font-size: 10px;
}
.frame-detail-label { color: #666; font-weight: 500; }
.frame-detail-value { color: #1f2937; font-weight: 600; }
.frame-example {
  background: white;
  border: 1px solid #e5e7eb;
  padding: 10px;
  border-radius: 4px;
  margin-top: 10px;
  font-size: 10px;
}
.frame-example-title { font-weight: 600; margin-bottom: 6px; }
.frame-example-row { display: flex; justify-content: space-between; margin: 3px 0; }
.overhead-percent { color: #059669; font-weight: 600; }
@media (max-width: 640px) {
  .frame-diagram { flex-direction: column; align-items: flex-start; }
}
</style>

<div class="frame-section">
  <div class="frame-scenarios">
    <div class="frame-scenario">
      <div class="frame-scenario-title">📋 Scenario 1: Minimum Frame (Unmasked, Server→Browser)</div>
      <div class="frame-diagram">
        <div class="frame-byte b0">
          Byte 0<br><div class="frame-label-small">FIN|RSV|Opcode</div>
        </div>
        <div class="frame-byte b1">
          Byte 1<br><div class="frame-label-small">MASK|Len</div>
        </div>
        <div style="color:#9ca3af; margin:0 4px;">→</div>
        <div class="frame-byte payload">
          Payload (320 bytes)<br><div class="frame-label-small">Audio chunk</div>
        </div>
      </div>
      <div class="frame-breakdown">
        <span class="frame-detail-label">Total header:</span>
        <span class="frame-detail-value">2 bytes</span>
      </div>
      <div class="frame-example">
        <div class="frame-example-title">Example: 320-byte audio from server to browser</div>
        <div class="frame-example-row">
          <span>Header size:</span>
          <span class="frame-detail-value">2 bytes</span>
        </div>
        <div class="frame-example-row">
          <span>Payload size:</span>
          <span class="frame-detail-value">320 bytes</span>
        </div>
        <div class="frame-example-row">
          <span>Overhead:</span>
          <span class="overhead-percent">0.6%</span>
        </div>
      </div>
    </div>

    <div class="frame-scenario">
      <div class="frame-scenario-title">📋 Scenario 2: Extended Frame (Masked, Browser→Server)</div>
      <div class="frame-diagram">
        <div class="frame-byte b0">Byte 0<br><div class="frame-label-small">FIN|RSV|Opcode</div></div>
        <div class="frame-byte b1">Byte 1<br><div class="frame-label-small">MASK|127</div></div>
        <div class="frame-byte b2-3">Bytes 2-3<br><div class="frame-label-small">Length</div></div>
        <div class="frame-byte b4-7">Bytes 4-7<br><div class="frame-label-small">Mask Key</div></div>
        <div style="color:#9ca3af; margin:0 4px;">→</div>
        <div class="frame-byte payload">Payload (320 bytes)</div>
      </div>
      <div class="frame-breakdown">
        <span class="frame-detail-label">Total header:</span>
        <span class="frame-detail-value">8 bytes (why? RFC requires browsers mask)</span>
      </div>
      <div class="frame-example">
        <div class="frame-example-title">Example: 320-byte audio from browser to server</div>
        <div class="frame-example-row">
          <span>Header size:</span>
          <span class="frame-detail-value">8 bytes</span>
        </div>
        <div class="frame-example-row">
          <span>Payload size:</span>
          <span class="frame-detail-value">320 bytes</span>
        </div>
        <div class="frame-example-row">
          <span>Overhead:</span>
          <span class="overhead-percent">2.5%</span>
        </div>
        <div class="frame-example-row" style="margin-top:6px; padding-top:6px; border-top:1px solid #e5e7eb; color:#666; font-size:9px;">
          <span><strong>Why mask?</strong> Prevent cache poisoning attack (proxy injection)</span>
        </div>
      </div>
    </div>

    <div class="frame-scenario">
      <div class="frame-scenario-title">📋 Scenario 3: 64-bit Length (Rare, >65KB payload)</div>
      <div class="frame-diagram">
        <div class="frame-byte b0">Byte 0-1<br><div class="frame-label-small">Header</div></div>
        <div class="frame-byte b2-3">Bytes 2-9<br><div class="frame-label-small">64-bit Length</div></div>
        <div class="frame-byte b4-7">Bytes 10-13<br><div class="frame-label-small">Mask Key</div></div>
        <div style="color:#9ca3af; margin:0 4px;">→</div>
        <div class="frame-byte payload">Payload</div>
      </div>
      <div class="frame-breakdown">
        <span class="frame-detail-label">Total header:</span>
        <span class="frame-detail-value">14 bytes</span>
      </div>
      <div style="font-size:10px; color:#666; margin-top:10px; padding:8px; background:white; border-radius:4px; border:1px solid #e5e7eb;">
        <strong>When used:</strong> Only for large payloads (>65,535 bytes). In Voice AI, payload typically 320 bytes, so scenario 1 or 2 only.
      </div>
    </div>
  </div>

  <div style="background:#fef3c7; border-left:3px solid #fbbf24; padding:12px; border-radius:4px; margin-top:15px; font-size:11px;">
    <strong>🔑 Key points:</strong>
    <ul style="margin:6px 0 0 0; padding-left:16px;">
      <li><strong>FIN bit:</strong> 1 = final fragment, 0 = more fragments coming (for big payloads)</li>
      <li><strong>Opcode:</strong> text (1) vs binary (2) vs close (8) vs ping (9) vs pong (10)</li>
      <li><strong>MASK bit:</strong> 1 = masking applied (browser→server must have), 0 = no mask (server→browser usually no mask)</li>
      <li><strong>Why 2-14 bytes?</strong> Depends on payload size + whether masked. Still 50×-100× less overhead than HTTP headers!</li>
    </ul>
  </div>
</div>

</div>

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

### P11 — Architecture Diagram: Browser vs Phone

![Browser Direct WebSocket (Opus 48kHz) vs Phone Double WebSocket (G.711 8kHz → Transcode → Opus)](./images/browser-vs-phone-architecture.png)

*Figure: Hai pattern WebSocket khác nhau trong Voice AI*

*Pattern 1 (Browser, trái): Audio đi thẳng browser ↔ AI qua single WebSocket, latency minimal (~100-150ms). Pattern 2 (Phone, phải): Audio từ PSTN qua Twilio → Server (transcode G.711→PCM→Opus) → AI Model qua 2 WebSockets, latency cao hơn (~150-250ms) vì transcode và session state management ở server.*

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

## Failure Modes & Recovery — Điều Gì Sẽ Sai Lệch

Đến đây, bạn hiểu cách WebSocket hoạt động ở happy path. Nhưng production không phải happy path. Dưới đây là failure scenarios thực tế mà bạn sẽ gặp:

### Scenario 1: AI Model Crashes Mid-Stream

**Điều gì xảy ra:**
1. User đang nói → audio frames đến server qua WebSocket
2. AI Model bất ngờ crash (OOM, NaN in tensor, timeout)
3. Server không send lại response frames
4. Client nhận không có audio, hung trên màn hình
5. Không auto-reconnect vì client không biết server đã chết

**Recovery Strategy:**
- **Server side:** Implement try-catch around AI inference. Nếu fail → close WebSocket với status `1011` (Server Error), không im lặng
- **Client side:** On receiving 1011 → auto-reconnect sau exponential backoff (50ms, 100ms, 200ms, etc.)
- **Timeout:** Nếu không nhận response audio trong 3 seconds → client assume server dead, close + reconnect
- **Logging:** Log mỗi unexpected close kèm error stack trace

```javascript
// Server pseudocode
try {
  const aiResponse = await aiModel.inference(userAudio);
  ws.send(aiResponse);
} catch (err) {
  console.error('AI inference failed:', err);
  ws.close(1011, 'Server error: ' + err.message);  // Explicit error code
}

// Client pseudocode
ws.onclose = (event) => {
  if (event.code === 1011) {
    reconnectWithBackoff();  // Exponential backoff
  }
};
```

### Scenario 2: Network Disconnection During Call

**Điều gì xảy ra:**
1. Phone call đang active
2. User đi qua vùng WiFi yếu → packet loss 20%+
3. WebSocket frames bị drop → audio gaps, choppy playback
4. Sau ~30 frames without pong → server close connection (1006 - abnormal closure)
5. Audio hoàn toàn bị cắt

**Recovery Strategy:**
- **Packet loss tolerance:** Buffer 3-5 frames (60-100ms) ở client để absorb jitter/loss
- **Jitter buffer:** Dynamic adjust dựa trên packet arrival variance
- **Graceful degradation:** Nếu packet loss >10%, reduce audio quality hoặc enable PLC (Packet Loss Concealment)
- **Server-side:** Don't immediately close on ping timeout, wait 2 timeouts (60s) trước closing
- **Client reconnect:** Exponential backoff + resume from last good state (không restart conversation)

```javascript
// Client jitter buffer pseudocode
class JitterBuffer {
  constructor(minSize = 3, maxSize = 10) {
    this.buffer = [];
    this.targetSize = minSize;
  }
  
  onFrameReceived(frame, delayMs) {
    this.buffer.push(frame);
    
    if (delayMs > this.targetSize * 20) {
      this.targetSize = Math.min(this.targetSize + 1, this.maxSize);  // Expand
    } else if (delayMs < this.targetSize * 20 / 2) {
      this.targetSize = Math.max(this.targetSize - 1, 1);  // Shrink
    }
  }
}
```

### Scenario 3: Server Restart / Graceful Shutdown

**Điều gì xảy ra:**
1. DevOps rolls new version → server start graceful shutdown
2. Server có 50 active WebSocket connections
3. If not handled → close immediately → 50 users: "Call dropped"
4. If handled → drain connections, close gracefully → users re-route to healthy instance

**Recovery Strategy:**
- **Graceful shutdown protocol:** On SIGTERM, server:
  1. Stop accepting new connections
  2. Set timeout for graceful drain (e.g., 30 seconds)
  3. Send close frame to all open WebSockets with code `1001` (Going Away)
  4. Wait for client acks, then exit
  5. If timeout reached → force close remaining
  
- **Client handling:** On receiving 1001 (Going Away) → client immediately reconnect to load balancer, which routes to healthy instance
- **Load balancer:** Connection draining aware. Route new calls away from draining server

```bash
# Server shutdown pseudocode
handle_sigterm:
  stop_accepting_new_connections()
  start_graceful_drain(timeout=30s):
    for ws in active_websockets:
      ws.close(1001, 'Server restarting')
    wait_for_acks(timeout=30s)
    if still_open(ws):
      ws.close_force()  # Force close after timeout
```

### Scenario 4: Zombie Connections (Resource Leak)

**Điều gì xảy ra:**
1. Client browser tab closed → browser forgot to close WebSocket
2. Server waiting for pong, never arrives → after 30s, server close
3. But if server doesn't check pong properly → connection persists
4. After 1000 such zombies → memory leak → server OOM → crash all users

**Recovery Strategy:**
- **Strict ping/pong enforcement:** Log every missed pong, close immediately on timeout
- **Connection timeout per user:** Set absolute max connection lifetime (e.g., 30 minutes). After that, force close + require re-auth
- **Memory monitoring:** Alert if per-connection memory > expected. Auto-kill zombie processes.
- **Browser cleanup:** Implement onbeforeunload to gracefully close WebSocket before page unload

```javascript
// Client cleanup
window.addEventListener('beforeunload', () => {
  if (ws.readyState === WebSocket.OPEN) {
    ws.close(1000, 'User navigating away');  // Graceful close
  }
});
```

### Scenario 5: Concurrent Calls from Same User

**Điều gì xảy ra:**
1. User opens two browser tabs, both call AI receptionist
2. Same ephemeral token used for both
3. Server routes both to same session
4. Audio from both tabs mixed → AI confused
5. Result: garbled output or one call dropped

**Recovery Strategy:**
- **Session per connection:** Each WebSocket connection = separate session, even if same token
- **Token rate limiting:** Limit how many WebSocket connections per token (e.g., max 1)
- **Concurrent call detection:** Server detects 2 connections from same token → close older one with code `1008` (Policy Violation)
- **Client warning:** Alert user "Call already in progress, disconnect old tab?"

---

## Best Practices Summary

| Scenario | What Breaks | Recovery |
|----------|------------|----------|
| **AI crashes** | Silent hang | Close 1011, client reconnect |
| **Network loss** | Audio dropout | Jitter buffer, PLC, graceful resume |
| **Server restart** | 50 call drops | Graceful drain 1001 code |
| **Zombie connections** | Memory leak | Strict ping/pong, auto-kill timeout |
| **Concurrent calls** | Audio mixing | Session per connection, rate limit |

### P8 — Connection Lifecycle Timeline: Happy Path + Failure Scenarios

<div class="html-component">
<style>
.timeline-section { font-family: "Segoe UI", sans-serif; }
.lifecycle-container { margin: 20px 0; }
.lifecycle-path {
  display: grid;
  gap: 20px;
  margin: 20px 0;
}
.lifecycle-row {
  border-left: 3px solid;
  padding: 15px;
  border-radius: 4px;
  background: #f9f9f9;
}
.lifecycle-row.happy { border-left-color: #1D9E75; background: #f0fdf4; }
.lifecycle-row.failure { border-left-color: #dc2626; background: #fef2f2; }
.lifecycle-label { font-weight: 600; font-size: 13px; margin-bottom: 10px; }
.lifecycle-flow {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-wrap: wrap;
  font-size: 11px;
}
.lc-box {
  background: white;
  border: 1px solid #e5e7eb;
  padding: 6px 10px;
  border-radius: 4px;
  white-space: nowrap;
  font-weight: 500;
}
.lc-box.state-opening { background: #dbeafe; color: #1e40af; border-color: #93c5fd; }
.lc-box.state-active { background: #dbeafe; color: #1e40af; border-color: #93c5fd; }
.lc-box.state-ping { background: #fef3c7; color: #92400e; border-color: #fcd34d; }
.lc-box.state-close { background: #e5e7eb; color: #374151; border-color: #d1d5db; }
.lc-box.state-error { background: #fee2e2; color: #991b1b; border-color: #fca5a5; }
.lc-arrow { color: #9ca3af; font-weight: bold; }
.lc-timing { font-size: 10px; color: #666; margin-top: 5px; }
.lc-details { font-size: 10px; color: #666; margin-top: 8px; line-height: 1.4; }
</style>

<div class="timeline-section">
  <div class="lifecycle-container">
    <div class="lifecycle-path">
      <div class="lifecycle-row happy">
        <div class="lifecycle-label">✅ Happy Path — Normal Connection</div>
        <div class="lifecycle-flow">
          <div class="lc-box state-opening">Opening<br>Handshake</div>
          <div class="lc-arrow">→</div>
          <div class="lc-box state-active">Active<br>(Frames)</div>
          <div class="lc-arrow">⟷</div>
          <div class="lc-box state-ping">Ping/Pong<br>Heartbeat</div>
          <div class="lc-arrow">→</div>
          <div class="lc-box state-close">Close<br>Frame</div>
          <div class="lc-arrow">→</div>
          <div class="lc-box state-close">TCP Close</div>
        </div>
        <div class="lc-timing">~100ms → 20ms-hours → 5-10s → 1s</div>
        <div class="lc-details">Handshake once, stream audio continuously, periodic heartbeat verifies connection alive, graceful close when done.</div>
      </div>

      <div class="lifecycle-row failure">
        <div class="lifecycle-label">❌ Failure #1 — Network Disconnection</div>
        <div class="lifecycle-flow">
          <div class="lc-box state-active">Active</div>
          <div class="lc-arrow">→</div>
          <div class="lc-box state-error">[Packet<br>Loss]</div>
          <div class="lc-arrow">→</div>
          <div class="lc-box state-error">Backoff<br>1s</div>
          <div class="lc-arrow">→</div>
          <div class="lc-box state-active">Reconnect</div>
          <div class="lc-arrow">→</div>
          <div class="lc-box state-active">Resume</div>
        </div>
        <div class="lc-timing">exponential backoff (1s, 2s, 4s, ...)</div>
        <div class="lc-details">Client detects packets missing. Solution: jitter buffer (3-5 frames), exponential backoff, resume from last good state (not restart).</div>
      </div>

      <div class="lifecycle-row failure">
        <div class="lifecycle-label">❌ Failure #2 — AI Model Crashes</div>
        <div class="lifecycle-flow">
          <div class="lc-box state-active">Active</div>
          <div class="lc-arrow">→</div>
          <div class="lc-box state-error">Model<br>Error 1011</div>
          <div class="lc-arrow">→</div>
          <div class="lc-box state-error">Client<br>Backoff</div>
          <div class="lc-arrow">→</div>
          <div class="lc-box state-active">Reconnect</div>
        </div>
        <div class="lc-timing">immediate 1011 response, exponential backoff</div>
        <div class="lc-details">Server wraps AI inference in try-catch, sends close code 1011 (Server Error) explicitly. Client recognizes 1011 = server error, auto-reconnect.</div>
      </div>

      <div class="lifecycle-row failure">
        <div class="lifecycle-label">❌ Failure #3 — Ping/Pong Timeout</div>
        <div class="lifecycle-flow">
          <div class="lc-box state-ping">Ping</div>
          <div class="lc-arrow">→</div>
          <div class="lc-box state-error">[No Pong<br>after 30s]</div>
          <div class="lc-arrow">→</div>
          <div class="lc-box state-error">Timeout</div>
          <div class="lc-arrow">→</div>
          <div class="lc-box state-close">Close</div>
          <div class="lc-arrow">→</div>
          <div class="lc-box state-active">Reconnect</div>
        </div>
        <div class="lc-timing">30s timeout, exponential backoff</div>
        <div class="lc-details">Server sends ping, waits for pong. If pong not received after 30s (typical), assume connection dead. Close and reconnect.</div>
      </div>

      <div class="lifecycle-row failure">
        <div class="lifecycle-label">❌ Failure #4 — Server Restart</div>
        <div class="lifecycle-flow">
          <div class="lc-box state-active">Active</div>
          <div class="lc-arrow">→</div>
          <div class="lc-box state-error">SIGTERM<br>(drain)</div>
          <div class="lc-arrow">→</div>
          <div class="lc-box state-close">Close 1001<br>(Going Away)</div>
          <div class="lc-arrow">→</div>
          <div class="lc-box state-active">Reconnect<br>to new server</div>
        </div>
        <div class="lc-timing">graceful drain 30s</div>
        <div class="lc-details">DevOps restart → server sends 1001 to all connections. Load balancer routes reconnect to healthy instance. Zero user impact if handled properly.</div>
      </div>
    </div>
  </div>
</div>

</div>

---

## So sánh WebSocket với giao thức realtime khác

Trước khi kết luận, hãy hiểu tại sao Voice AI chọn WebSocket thay vì các lựa chọn khác:

### WebRTC (Web Real-Time Communication)

Giao thức **peer-to-peer** cho audio/video realtime trên browser.

**Đặc điểm:**
- Tối ưu cho media (codec negotiation, echo cancellation, bandwidth adaptation)
- Peer-to-peer, không cần server trung gian cho media
- Native trên browser (tất cả browsers support)

**So với WebSocket:**
- WebRTC: Media heavy, signaling light → cho pair-to-peer voice calls
- WebSocket: Protocol agnostic → cho bất kỳ data nào

**Trong Voice AI:**
- Browser channel: WebRTC cho audio (P2P), WebSocket cho signaling (control messages)
- Phone channel: Không thể dùng WebRTC (PSTN không support), chỉ WebSocket + Twilio gateway
- Kết hợp: Browser uses both; Phone uses WebSocket only

**Khi nào dùng pure WebRTC?** Nếu không có server inference (peer-to-peer AIs), WebRTC đủ. Nhưng Voice AI cần server, nên cần WebSocket cho control.

---

### gRPC Streaming

Framework RPC hỗ trợ bidirectional streaming qua HTTP/2.

**Ưu điểm:**
- Typed contracts (protobuf) → type safety
- Built-in load balancing, health checks
- Compact binary protocol
- Multi-language support

**Nhược điểm:**
- Phức tạp: Cần certificate, proto compiler setup
- Không native trên browser (cần gRPC-web proxy, thêm latency)
- Overkill cho Voice AI (bạn không cần typed RPC, chỉ audio streaming)
- Latency impact: gRPC-web proxy = +10-30ms

**Khi nào dùng gRPC?** Khi bạn có enterprise ecosystem với gRPC backend + typed APIs cần bảo vệ. Voice AI là use case quá specific.

---

### MQTT

Giao thức pub/sub nhẹ, phổ biến trong IoT.

**Ưu điểm:**
- Lightweight (design cho embedded devices)
- Good for pub/sub patterns (bạn publish, multiple subscribers receive)

**Nhược điểm:**
- Thiết kế cho message passing (discrete messages), không tối ưu cho continuous streaming
- Publish-subscribe model doesn't fit voice call (không phải 1→N, là 1↔1)
- Not native on browser
- Overkill cho Voice AI

**Khi nào dùng MQTT?** Khi bạn cần sensor data → multiple dashboards (IoT). Không phải voice.

---

### 🎯 **Decision Matrix: Khi nào dùng gì**

| Yêu cầu | WebSocket | WebRTC | gRPC | MQTT |
|---------|-----------|--------|------|------|
| **Continuous audio streaming** | ✅ Tối ưu | ✅ Tối ưu | ⚠️ Overkill | ❌ Sai |
| **Browser support native** | ✅ Yes | ✅ Yes | ❌ (cần proxy) | ❌ No |
| **PSTN integration** | ✅ (qua Twilio) | ❌ No | ❌ No | ❌ No |
| **Low latency** | ✅ <200ms | ✅ <150ms | ⚠️ +30ms overhead | ❌ Not designed |
| **Bidirectional** | ✅ Yes | ✅ Yes (P2P) | ✅ Yes | ⚠️ (pub/sub only) |
| **Type safety** | ❌ No | ❌ No | ✅ Yes | ❌ No |
| **Complexity** | ✅ Simple | ⚠️ Medium | ❌ Complex | ⚠️ Medium |
| **Ecosystem** | ✅ Mature | ✅ Mature | ⚠️ Enterprise | ❌ IoT-focused |

**Recommendation logic:**
```
IF need to accept phone calls (PSTN) → MUST use WebSocket (Twilio only supports WS)
ELSE IF need peer-to-peer voice (no server) → use WebRTC
ELSE IF need typed RPC contracts + load balancing → use gRPC
ELSE IF building IoT sensor network → use MQTT
ELSE (Voice AI with server) → use WebSocket + (WebRTC for browser audio)
```

---

### 📍 **Voice AI Default: WebSocket (+ WebRTC for browser)**

Kenapa? 
- Simplicity wins in production (less moving parts = fewer bugs)
- WebSocket supports both Browser (via Twilio signaling) + Phone (via Twilio audio)
- Works everywhere (native browser support, native PSTN support)
- Low latency, low overhead
- Proven at scale (used by Twilio, OpenAI Realtime, etc.)

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

## 💡 Latency Budget — WebSocket chỉ là 2% của vấn đề

Nhiều developer tin rằng: "WebSocket slow, phải optimize protocol."

**Sai.** Điều đó cho thấy hiểu lầm cơ bản về latency budget của Voice AI.

### Latency Budget toàn bộ hệ thống: 300ms

Một cuộc gọi Voice AI điển hình có tổng latency ~300ms trước khi user cảm thấy "slow." Breakdown:

<div class="html-component">
<style>
.latency-section { padding: 20px; font-family: "Segoe UI", sans-serif; color: #333; }
.budget-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; margin: 20px 0; }
.budget-item { 
  padding: 15px; 
  border-left: 4px solid; 
  background: #f9f9f9; 
  border-radius: 4px;
}
.budget-item.websocket { border-left-color: #534AB7; }
.budget-item.network { border-left-color: #378ADD; }
.budget-item.ai { border-left-color: #1D9E75; }
.budget-item.tls { border-left-color: #FFB81C; }
.budget-label { font-weight: 600; margin-bottom: 5px; }
.budget-time { font-size: 24px; color: #534AB7; font-weight: 700; }
.budget-item.network .budget-time { color: #378ADD; }
.budget-item.ai .budget-time { color: #1D9E75; }
.budget-item.tls .budget-time { color: #FFB81C; }
.budget-note { font-size: 12px; color: #666; margin-top: 8px; }

.budget-visual {
  margin: 30px 0;
  padding: 20px;
  background: linear-gradient(135deg, #f5f5f5 0%, #ffffff 100%);
  border-radius: 8px;
  text-align: center;
}
.budget-bar {
  display: flex;
  height: 50px;
  border-radius: 4px;
  overflow: hidden;
  gap: 2px;
  background: white;
  margin: 15px 0;
  box-shadow: inset 0 1px 2px rgba(0,0,0,0.1);
}
.budget-segment {
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-weight: 600;
  font-size: 11px;
  text-shadow: 0 1px 2px rgba(0,0,0,0.3);
}
.segment-ws { background: #534AB7; width: 2%; }
.segment-tls { background: #FFB81C; width: 8%; }
.segment-network { background: #378ADD; width: 35%; }
.segment-ai { background: #1D9E75; width: 55%; }

.budget-legend {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 20px;
  margin-top: 15px;
  font-size: 12px;
}
.legend-item {
  display: flex;
  align-items: center;
  gap: 8px;
}
.legend-color {
  width: 20px;
  height: 20px;
  border-radius: 3px;
}
</style>

<div class="latency-section">
  <div class="budget-visual">
    <h3 style="margin-top: 0;">Latency Budget 300ms — Breakdown thực tế</h3>
    
    <div class="budget-bar">
      <div class="budget-segment segment-ws" title="WebSocket">WS 2%</div>
      <div class="budget-segment segment-tls" title="TLS">TLS 8%</div>
      <div class="budget-segment segment-network" title="Network">Network 35%</div>
      <div class="budget-segment segment-ai" title="AI Processing">AI 55%</div>
    </div>
    
    <div class="budget-legend">
      <div class="legend-item">
        <div class="legend-color" style="background: #534AB7;"></div>
        <span>WebSocket 6ms</span>
      </div>
      <div class="legend-item">
        <div class="legend-color" style="background: #FFB81C;"></div>
        <span>TLS 24ms</span>
      </div>
      <div class="legend-item">
        <div class="legend-color" style="background: #378ADD;"></div>
        <span>Network 105ms</span>
      </div>
      <div class="legend-item">
        <div class="legend-color" style="background: #1D9E75;"></div>
        <span>AI 165ms</span>
      </div>
    </div>
  </div>

  <div class="budget-grid">
    <div class="budget-item websocket">
      <div class="budget-label">🔌 WebSocket Frame Overhead</div>
      <div class="budget-time">~6ms</div>
      <div class="budget-note">
        Handshake + frame overhead (2-14 bytes/frame). One-time cost (~100-150ms handshake) amortized qua 50 chunks/giây → negligible.
      </div>
    </div>

    <div class="budget-item tls">
      <div class="budget-label">🔒 TLS Encryption</div>
      <div class="budget-time">~24ms</div>
      <div class="budget-note">
        One-time TLS handshake (50-100ms) amortized qua call duration. Encryption/decryption per-frame: &lt;0.5ms.
      </div>
    </div>

    <div class="budget-item network">
      <div class="budget-label">🌐 Network Round-Trip (RTT)</div>
      <div class="budget-time">~105ms</div>
      <div class="budget-note">
        Audio từ user → Server → AI Model → Response ngược lại. Typically 30-50ms each way, plus queuing + jitter buffer = 80-150ms.
      </div>
    </div>

    <div class="budget-item ai">
      <div class="budget-label">🧠 AI Processing</div>
      <div class="budget-time">~165ms</div>
      <div class="budget-note">
        STT (speech-to-text) 40-60ms + LLM inference 80-120ms + TTS (text-to-speech) 30-40ms. <strong>Tầng mà bạn có thể optimize nhất.</strong>
      </div>
    </div>
  </div>
</div>

### ⚠️ **Redis Sync Catastrophe** — Tại sao Unified Server Pattern quan trọng

Một số architect cố gắng chia nhỏ thành microservices rồi **sync state qua Redis mỗi 20ms:**

```
Browser → Server #1 (receives audio)
          ↓ PUT to Redis (network call)
          Server #2 (AI model)
          ↓ GET from Redis (network call)
          → Send response to Browser
```

**Kết quả:**

| Thành phần | Latency | % Budget |
|-----------|---------|----------|
| Network RTT | 30ms | 10% |
| Redis PUT | 15-30ms | 5-10% |
| Redis GET | 15-30ms | 5-10% |
| **Total overhead** | **60-90ms** | **20-30% budget** |

**Nếu Redis overhead là 60-90ms, thì AI processing bị cắt từ 165ms xuống còn 75-105ms.** Voice AI chất lượng sụt 40%. Đó là lý do **Bài 12 (Unified Server Pattern)** nhắn: "Không dùng Redis cho realtime state sync — tất cả state (session, buffer, interrupt flags) phải nằm **cùng process** (shared memory)."

### Implication cho kiến trúc của bạn

**Nếu bạn optimize WebSocket tiếp theo, bạn chỉ giành được ~2% cải thiện.**  
**Nếu bạn optimize AI pipeline, bạn có thể giành 10-20% cải thiện.**  
**Nếu bạn tránh Redis sync, bạn tiết kiệm 20-30% ngay lập tức.**

Điều này giải thích tại sao startup demo (280ms response) thất bại lúc production — Redis overhead + network jitter + model latency dồn dập.

### P12 — Latency Budget Pie Chart: Visual Alternative

<div class="html-component">
<style>
.pie-section { font-family: "Segoe UI", sans-serif; margin: 20px 0; }
.pie-container {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 20px;
  margin: 20px 0;
  align-items: center;
}
.pie-chart {
  width: 200px;
  height: 200px;
  border-radius: 50%;
  background: conic-gradient(
    #1D9E75 0deg 198deg,
    #378ADD 198deg 324deg,
    #FFB81C 324deg 352deg,
    #534AB7 352deg 360deg
  );
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 0 auto;
  position: relative;
  font-size: 11px;
  font-weight: 600;
  color: white;
  text-shadow: 0 1px 2px rgba(0,0,0,0.3);
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
}
.pie-label {
  position: absolute;
  font-size: 10px;
  font-weight: 600;
  color: white;
  text-shadow: 0 1px 2px rgba(0,0,0,0.3);
}
.pie-legend {
  display: flex;
  flex-direction: column;
  gap: 10px;
}
.pie-item {
  display: flex;
  align-items: flex-start;
  gap: 10px;
  padding: 8px;
  border-radius: 4px;
  background: #f9f9f9;
  font-size: 11px;
}
.pie-color {
  width: 16px;
  height: 16px;
  border-radius: 3px;
  flex-shrink: 0;
  margin-top: 2px;
}
.pie-info {
  flex: 1;
}
.pie-name { font-weight: 600; color: #1f2937; }
.pie-metric { color: #666; font-size: 10px; }
.pie-percent { font-weight: 700; color: #1f2937; }
.pie-breakdown {
  margin-top: 20px;
  padding: 15px;
  background: #f0fdf4;
  border-left: 3px solid #1D9E75;
  border-radius: 4px;
  font-size: 11px;
}
.pie-breakdown-title { font-weight: 600; margin-bottom: 8px; color: #1f2937; }
.pie-breakdown-item {
  display: flex;
  justify-content: space-between;
  margin: 5px 0;
  padding: 5px 0;
  border-bottom: 1px solid rgba(0,0,0,0.05);
}
.pie-breakdown-label { color: #666; }
.pie-breakdown-value { font-weight: 600; color: #1f2937; }
.redis-warning {
  margin-top: 15px;
  padding: 15px;
  background: #fef2f2;
  border-left: 3px solid #dc2626;
  border-radius: 4px;
  font-size: 11px;
}
.redis-warning-title { font-weight: 600; margin-bottom: 8px; color: #991b1b; }
.redis-warning-text { color: #666; line-height: 1.5; margin-bottom: 8px; }
.redis-impact {
  background: white;
  padding: 8px;
  border-radius: 4px;
  margin-top: 8px;
  border: 1px solid #fca5a5;
}
.redis-impact-row {
  display: flex;
  justify-content: space-between;
  margin: 4px 0;
  font-size: 10px;
}
.redis-impact-label { color: #666; }
.redis-impact-value { font-weight: 600; color: #dc2626; }
@media (max-width: 768px) {
  .pie-container { grid-template-columns: 1fr; }
  .pie-chart { width: 160px; height: 160px; }
}
</style>

<div class="pie-section">
  <div class="pie-container">
    <div>
      <div class="pie-chart">
        <div style="text-align:center;">
          <div style="font-size:20px; font-weight:700;">300ms</div>
          <div style="font-size:10px; margin-top:4px;">Total Budget</div>
        </div>
      </div>
    </div>

    <div class="pie-legend">
      <div class="pie-item">
        <div class="pie-color" style="background:#1D9E75;"></div>
        <div class="pie-info">
          <div class="pie-name">🧠 AI Processing</div>
          <div class="pie-metric"><span class="pie-percent">165ms</span> (55%)</div>
          <div class="pie-metric" style="color:#059669; margin-top:2px;">STT (40-60ms) + LLM (80-120ms) + TTS (30-40ms)</div>
        </div>
      </div>

      <div class="pie-item">
        <div class="pie-color" style="background:#378ADD;"></div>
        <div class="pie-info">
          <div class="pie-name">🌐 Network RTT</div>
          <div class="pie-metric"><span class="pie-percent">105ms</span> (35%)</div>
          <div class="pie-metric" style="color:#0ea5e9; margin-top:2px;">Client→Server→AI (30-50ms each) + jitter + queuing</div>
        </div>
      </div>

      <div class="pie-item">
        <div class="pie-color" style="background:#FFB81C;"></div>
        <div class="pie-info">
          <div class="pie-name">🔒 TLS Overhead</div>
          <div class="pie-metric"><span class="pie-percent">24ms</span> (8%)</div>
          <div class="pie-metric" style="color:#f59e0b; margin-top:2px;">One-time handshake (50-100ms) amortized per call</div>
        </div>
      </div>

      <div class="pie-item">
        <div class="pie-color" style="background:#534AB7;"></div>
        <div class="pie-info">
          <div class="pie-name">🔌 WebSocket</div>
          <div class="pie-metric"><span class="pie-percent">6ms</span> (2%)</div>
          <div class="pie-metric" style="color:#7c3aed; margin-top:2px;">Frame overhead (2-14 bytes) + handshake amortized</div>
        </div>
      </div>
    </div>
  </div>

  <div class="pie-breakdown">
    <div class="pie-breakdown-title">📊 Budget Allocation — Nơi bạn nên focus optimize</div>
    <div class="pie-breakdown-item">
      <span class="pie-breakdown-label"><strong>#1 Priority:</strong> AI Processing</span>
      <span class="pie-breakdown-value">165ms (55%)</span>
    </div>
    <div class="pie-breakdown-item">
      <span class="pie-breakdown-label">Use streaming STT (Deepgram, OpenAI Whisper)</span>
      <span class="pie-breakdown-value">saves 10-20ms</span>
    </div>
    <div class="pie-breakdown-item">
      <span class="pie-breakdown-label">Use streaming TTS (ElevenLabs, Google Cloud)</span>
      <span class="pie-breakdown-value">saves 5-10ms</span>
    </div>
    <div class="pie-breakdown-item">
      <span class="pie-breakdown-label"><strong>#2 Priority:</strong> Network RTT</span>
      <span class="pie-breakdown-value">105ms (35%)</span>
    </div>
    <div class="pie-breakdown-item">
      <span class="pie-breakdown-label">Use CDN for nearest region (hard to improve)</span>
      <span class="pie-breakdown-value">~10-20ms max</span>
    </div>
    <div class="pie-breakdown-item">
      <span class="pie-breakdown-label"><strong>#3 Priority:</strong> TLS Overhead</span>
      <span class="pie-breakdown-value">24ms (8%)</span>
    </div>
    <div class="pie-breakdown-item">
      <span class="pie-breakdown-label">Use connection pooling (one-time cost)</span>
      <span class="pie-breakdown-value">negligible</span>
    </div>
    <div class="pie-breakdown-item">
      <span class="pie-breakdown-label"><strong>#4 Priority:</strong> WebSocket</span>
      <span class="pie-breakdown-value">6ms (2%)</span>
    </div>
    <div class="pie-breakdown-item">
      <span class="pie-breakdown-label">Optimize frame framing (tiny gain)</span>
      <span class="pie-breakdown-value">&lt;1ms</span>
    </div>
  </div>

  <div class="redis-warning">
    <div class="redis-warning-title">⚠️ CRITICAL: Redis Sync Catastrophe (Reviewed above)</div>
    <div class="redis-warning-text">
      Nếu bạn dùng microservices + Redis sync state mỗi 20ms để synchronize session info giữa các services:
    </div>
    <div class="redis-impact">
      <div class="redis-impact-row">
        <span class="redis-impact-label">Redis PUT latency:</span>
        <span class="redis-impact-value">15-30ms</span>
      </div>
      <div class="redis-impact-row">
        <span class="redis-impact-label">Redis GET latency:</span>
        <span class="redis-impact-value">15-30ms</span>
      </div>
      <div class="redis-impact-row">
        <span class="redis-impact-label">Total overhead:</span>
        <span class="redis-impact-value">60-90ms</span>
      </div>
      <div class="redis-impact-row">
        <span class="redis-impact-label">AI Processing shrinks from:</span>
        <span class="redis-impact-value">165ms → 75-105ms</span>
      </div>
      <div class="redis-impact-row" style="margin-top:6px; padding-top:6px; border-top:1px solid #fca5a5;">
        <span class="redis-impact-label"><strong>Quality loss:</strong></span>
        <span class="redis-impact-value">~40% worse</span>
      </div>
    </div>
    <div style="margin-top:8px; font-size:10px; color:#991b1b;">
      <strong>Solution:</strong> Unified Server Pattern (Bài 12) — keep tất cả state (session, buffers, interrupt flags) trong **shared memory** (cùng process), không sync qua network.
    </div>
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
