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
| Human agent | $4 – 10 | $40.000 – $100.000/ngày |
| AI (cascaded, GPT-4o) | $0.10 – 0.22 | $1.000 – $2.200/ngày |
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

<div class="component-embed">

<!-- architecture_4_layers.html -->

<h2 class="sr-only">Kiến trúc 4 tầng của hệ thống Voice AI production</h2>
<style>
  .arch-wrap{font-family:system-ui,-apple-system,'Segoe UI',sans-serif;padding:.5rem 0;color:#e2e8f0}
  .arch-layer{border-radius:10px;padding:14px 16px;margin:0 0 4px}
  .arch-layer-label{font-size:10px;font-weight:700;letter-spacing:.09em;text-transform:uppercase;margin:0 0 10px}

  /* CLIENT — blue tint */
  .layer-client{background:#182438}
  .layer-client .arch-layer-label{color:#60a5fa}
  /* GATEWAY — purple tint */
  .layer-gateway{background:#1c1830}
  .layer-gateway .arch-layer-label{color:#a78bfa}
  /* AI CORE — green tint + border */
  .layer-core{background:#14231e;border:.5px solid rgba(16,185,129,.35)}
  .layer-core .arch-layer-label{color:#34d399}
  /* DATA — neutral dark */
  .layer-data{background:#182030}
  .layer-data .arch-layer-label{color:#94a3b8}

  /* Cards inside layers */
  .client-channels,.gateway-items{display:grid;grid-template-columns:1fr 1fr;gap:8px}
  .channel-card,.gw-item,.data-svc{background:#0f172a;border-radius:8px;padding:10px 12px;border:.5px solid #2d3f55}

  .channel-header{display:flex;align-items:center;gap:6px;margin:0 0 6px}
  .channel-icon{font-size:14px;line-height:1}
  .channel-name{font-size:12px;font-weight:600;color:#e2e8f0}
  .channel-badges{display:flex;gap:4px;flex-wrap:wrap}
  .badge{font-size:10px;font-family:ui-monospace,'Cascadia Code',monospace;padding:2px 6px;border-radius:4px;font-weight:600}
  .badge-blue{background:rgba(59,130,246,.18);color:#60a5fa}
  .badge-orange{background:rgba(249,115,22,.18);color:#fb923c}
  .badge-gray{background:#1e293b;color:#64748b}

  .gw-name{font-size:12px;font-weight:600;color:#e2e8f0;margin:0 0 4px}
  .gw-detail{font-size:11px;color:#64748b;font-family:ui-monospace,'Cascadia Code',monospace}

  /* AI Core internals */
  .core-model{background:#0f172a;border-radius:8px;padding:12px 14px;border:.5px solid rgba(52,211,153,.3);margin:0 0 8px}
  .core-model-name{font-size:13px;font-weight:600;color:#e2e8f0;margin:0 0 6px}
  .core-io{display:flex;align-items:center;gap:8px;margin:0 0 4px;flex-wrap:wrap}
  .core-io-badge{font-size:11px;font-family:ui-monospace,'Cascadia Code',monospace;padding:3px 8px;border-radius:4px;font-weight:600}
  .io-in{background:rgba(59,130,246,.18);color:#60a5fa}
  .io-out{background:rgba(16,185,129,.18);color:#34d399}
  .io-arrow{color:#475569;font-size:12px}
  .core-sub{font-size:11px;color:#64748b}
  .core-services{display:grid;grid-template-columns:repeat(3,1fr);gap:6px}
  .svc-chip{background:#0f172a;border-radius:6px;padding:8px 10px;border:.5px solid #2d3f55;text-align:center}
  .svc-name{font-size:11px;font-weight:500;color:#cbd5e1}
  .svc-desc{font-size:10px;color:#475569;margin:2px 0 0}

  /* Data layer */
  .data-top{display:grid;grid-template-columns:repeat(3,1fr);gap:8px}
  .data-svc-name{font-size:12px;font-weight:600;color:#e2e8f0;margin:0 0 5px}
  .data-svc-items{display:flex;flex-direction:column;gap:2px}
  .data-svc-item{font-size:10px;color:#64748b}

  /* Connector arrows */
  .arch-arrow-row{display:flex;justify-content:space-around;padding:2px 60px;margin:0}
  .arch-arrow{display:flex;flex-direction:column;align-items:center}
  .arch-arrow-line,.arch-merge-line{width:1px;height:16px;background:#334155}
  .arch-arrow-head,.arch-merge-head{width:0;height:0;border-left:4px solid transparent;border-right:4px solid transparent;border-top:5px solid #334155}
  .arch-merge-row{display:flex;justify-content:center;margin:0}
  .arch-merge{display:flex;flex-direction:column;align-items:center}

  .arch-note{font-size:11px;color:#475569;text-align:center;margin:10px 0 0;font-style:italic}

  @media(max-width:520px){
    .client-channels,.gateway-items,.core-services,.data-top{grid-template-columns:1fr}
    .arch-arrow-row{padding:2px 30px}
  }
</style>
<div class="arch-wrap">

  <div class="arch-layer layer-client">
    <div class="arch-layer-label">Client Layer</div>
    <div class="client-channels">
      <div class="channel-card">
        <div class="channel-header">
          <span class="channel-icon">🌐</span>
          <span class="channel-name">Browser</span>
        </div>
        <div class="channel-badges">
          <span class="badge badge-blue">WebRTC</span>
          <span class="badge badge-blue">Opus</span>
          <span class="badge badge-blue">48 kHz</span>
        </div>
      </div>
      <div class="channel-card">
        <div class="channel-header">
          <span class="channel-icon">📞</span>
          <span class="channel-name">Phone</span>
        </div>
        <div class="channel-badges">
          <span class="badge badge-orange">PSTN</span>
          <span class="badge badge-orange">G.711</span>
          <span class="badge badge-orange">8 kHz</span>
          <span class="badge badge-gray">via Twilio</span>
        </div>
      </div>
    </div>
  </div>

  <div class="arch-arrow-row">
    <div class="arch-arrow"><div class="arch-arrow-line"></div><div class="arch-arrow-head"></div></div>
    <div class="arch-arrow"><div class="arch-arrow-line"></div><div class="arch-arrow-head"></div></div>
  </div>

  <div class="arch-layer layer-gateway">
    <div class="arch-layer-label">Gateway Layer</div>
    <div class="gateway-items">
      <div class="gw-item">
        <div class="gw-name">Token API</div>
        <div class="gw-detail">ephemeral key · 30min TTL</div>
      </div>
      <div class="gw-item">
        <div class="gw-name">WSS Stream</div>
        <div class="gw-detail">Twilio / Telnyx audio</div>
      </div>
    </div>
  </div>

  <div class="arch-merge-row">
    <div class="arch-merge"><div class="arch-merge-line"></div><div class="arch-merge-head"></div></div>
  </div>

  <div class="arch-layer layer-core">
    <div class="arch-layer-label">AI Core</div>
    <div class="core-model">
      <div class="core-model-name">Real-time Voice Model</div>
      <div class="core-io">
        <span class="core-io-badge io-in">PCM 16 kHz in</span>
        <span class="io-arrow">→</span>
        <span class="core-io-badge io-out">PCM 24 kHz out</span>
      </div>
      <div class="core-sub">sub-300ms latency · native audio processing</div>
    </div>
    <div class="core-services">
      <div class="svc-chip">
        <div class="svc-name">Template Service</div>
        <div class="svc-desc">persona · prompt</div>
      </div>
      <div class="svc-chip">
        <div class="svc-name">Session Service</div>
        <div class="svc-desc">state · shutdown</div>
      </div>
      <div class="svc-chip">
        <div class="svc-name">Observability</div>
        <div class="svc-desc">P95 · WER · logs</div>
      </div>
    </div>
  </div>

  <div class="arch-merge-row">
    <div class="arch-merge"><div class="arch-merge-line"></div><div class="arch-merge-head"></div></div>
  </div>

  <div class="arch-layer layer-data">
    <div class="arch-layer-label">Application &amp; Data Layer</div>
    <div class="data-top">
      <div class="data-svc">
        <div class="data-svc-name">Security Layer</div>
        <div class="data-svc-items">
          <span class="data-svc-item">PII masking</span>
          <span class="data-svc-item">Guardrails</span>
        </div>
      </div>
      <div class="data-svc">
        <div class="data-svc-name">Redis Cache</div>
        <div class="data-svc-items">
          <span class="data-svc-item">Bridge state</span>
          <span class="data-svc-item">Semantic cache</span>
        </div>
      </div>
      <div class="data-svc">
        <div class="data-svc-name">PostgreSQL</div>
        <div class="data-svc-items">
          <span class="data-svc-item">Sessions</span>
          <span class="data-svc-item">Transcripts</span>
        </div>
      </div>
    </div>
  </div>

  <p class="arch-note">2 channel đầu vào khác nhau — hội tụ về cùng 1 AI Core · Channel chỉ là adapter</p>
</div>

</div>

**Insight cốt lõi: Channel chỉ là adapter — Core mới là bộ não.** Browser và Phone là 2 thế giới hoàn toàn khác nhau về codec, protocol, và latency profile. Nhưng cả hai đều phải được chuẩn hóa về PCM trước khi AI Core xử lý. Phần lớn bug production nằm ở lớp adapter này — không phải ở Core.

---

## Audio Pipeline — Nguyên tắc bất đối xứng

Thành phần mà hầu hết tutorial bỏ qua hoàn toàn.

<div class="component-embed">

<!-- inbound_outbound_timeline.html -->

<h2 class="sr-only">Inbound và Outbound audio pipeline trong Voice AI — nguyên tắc bất đối xứng</h2>
<style>
  .io-wrap{font-family:system-ui,-apple-system,'Segoe UI',sans-serif;padding:.5rem 0;color:#e2e8f0}
  .io-section-title{font-size:14px;font-weight:500;color:#e2e8f0;margin:0 0 12px}

  .dir-header{display:flex;align-items:center;gap:8px;margin:0 0 8px}
  .dir-badge{font-size:11px;font-weight:700;padding:3px 10px;border-radius:20px;letter-spacing:.04em}
  .dir-badge-in{background:rgba(59,130,246,.2);color:#60a5fa}
  .dir-badge-out{background:rgba(249,115,22,.2);color:#fb923c}
  .dir-label{font-size:12px;color:#94a3b8}

  .pipe-row{display:flex;align-items:stretch;gap:0;margin:0 0 6px;overflow-x:auto}
  .pipe-step{flex:1;min-width:72px;text-align:center;padding:10px 6px;font-size:11px;color:#94a3b8;line-height:1.4;border-radius:8px;background:#1e293b}
  .pipe-step strong{display:block;font-size:12px;color:#e2e8f0;font-weight:600;margin:0 0 2px}
  .step-source{background:#182438}
  .step-source-out{background:#14231e}
  .step-dest{background:#14231e}
  .step-buffer{background:rgba(234,179,8,.14);border:.5px solid rgba(234,179,8,.4)}
  .step-immediate{background:rgba(249,115,22,.14);border:.5px solid rgba(249,115,22,.4)}
  .pipe-arr{display:flex;align-items:center;padding:0 3px;flex-shrink:0;color:#475569;font-size:13px}

  .buffer-tag{display:inline-block;font-size:9px;font-weight:700;letter-spacing:.04em;padding:1px 5px;border-radius:3px;background:rgba(234,179,8,.25);color:#fbbf24;margin:3px 0 0;text-transform:uppercase}
  .immediate-tag{display:inline-block;font-size:9px;font-weight:700;letter-spacing:.04em;padding:1px 5px;border-radius:3px;background:rgba(249,115,22,.25);color:#fb923c;margin:3px 0 0;text-transform:uppercase}

  .asym-rule{display:grid;grid-template-columns:1fr 1fr;gap:8px;margin:12px 0 0}
  .asym-card{border-radius:8px;padding:10px 12px;border:.5px solid #334155}
  .asym-card-in{border-color:rgba(59,130,246,.4)}
  .asym-card-out{border-color:rgba(249,115,22,.4)}
  .asym-rule-label{font-size:11px;font-weight:600;margin:0 0 4px}
  .asym-rule-label-in{color:#60a5fa}
  .asym-rule-label-out{color:#fb923c}
  .asym-rule-text{font-size:11px;color:#94a3b8;line-height:1.5}

  .note-box{background:rgba(59,130,246,.1);border-radius:8px;padding:10px 14px;font-size:12px;color:#93c5fd;line-height:1.5;margin:12px 0 0}

  @media(max-width:520px){
    .pipe-step{min-width:60px;padding:8px 4px}
    .asym-rule{grid-template-columns:1fr}
  }
</style>
<div class="io-wrap">
  <div class="io-section-title">Audio pipeline — 2 chiều, 2 nguyên tắc khác nhau</div>

  <div class="dir-header">
    <span class="dir-badge dir-badge-in">INBOUND</span>
    <span class="dir-label">Phone → AI · có thể buffer</span>
  </div>
  <div class="pipe-row">
    <div class="pipe-step step-source"><strong>Phone</strong>G.711 µ-law<br>8 kHz · 8-bit</div>
    <div class="pipe-arr">→</div>
    <div class="pipe-step"><strong>Decode</strong>G.711 →<br>Linear PCM</div>
    <div class="pipe-arr">→</div>
    <div class="pipe-step"><strong>Upsample</strong>8 kHz →<br>16 kHz</div>
    <div class="pipe-arr">→</div>
    <div class="pipe-step step-buffer"><strong>Buffer</strong>~60 ms chunks<br><span class="buffer-tag">imperceptible</span></div>
    <div class="pipe-arr">→</div>
    <div class="pipe-step step-dest"><strong>AI Model</strong>PCM 16 kHz<br>input</div>
  </div>

  <div class="dir-header" style="margin-top:14px">
    <span class="dir-badge dir-badge-out">OUTBOUND</span>
    <span class="dir-label">AI → Phone · zero buffer</span>
  </div>
  <div class="pipe-row">
    <div class="pipe-step step-source-out"><strong>AI Model</strong>PCM 24 kHz<br>output</div>
    <div class="pipe-arr">→</div>
    <div class="pipe-step"><strong>Low-pass</strong>Anti-alias<br>filter</div>
    <div class="pipe-arr">→</div>
    <div class="pipe-step"><strong>Downsample</strong>24 kHz →<br>8 kHz</div>
    <div class="pipe-arr">→</div>
    <div class="pipe-step"><strong>Encode</strong>PCM →<br>G.711 µ-law</div>
    <div class="pipe-arr">→</div>
    <div class="pipe-step step-immediate"><strong>Phone</strong>8 kHz<br><span class="immediate-tag">immediate</span></div>
  </div>

  <div class="asym-rule">
    <div class="asym-card asym-card-in">
      <div class="asym-rule-label asym-rule-label-in">Inbound — buffer được</div>
      <div class="asym-rule-text">60ms nằm dưới ngưỡng perception (~80ms). Gom đủ chunk trước khi xử lý giúp model chính xác hơn.</div>
    </div>
    <div class="asym-card asym-card-out">
      <div class="asym-rule-label asym-rule-label-out">Outbound — zero tolerance</div>
      <div class="asym-rule-text">Bất kỳ delay nào ở output đều cảm nhận được ngay lập tức. Buffer output = tiếng nói bị "giật".</div>
    </div>
  </div>

  <div class="note-box">Lỗi phổ biến: apply cùng buffer strategy cho cả 2 chiều. Kết quả là AI nghe "lag" dù metrics P95 vẫn ổn — vì buffer outbound không xuất hiện trong latency measurement.</div>
</div>

</div>

---

## Ephemeral Token — Tại sao API key không được vào browser

Với channel browser, có một vấn đề bảo mật cơ bản: **WebSocket cần kết nối thẳng đến AI endpoint — nhưng không thể đặt API key trong client code vì bất kỳ ai cũng đọc được.**

Pattern giải quyết bằng 2 plane tách biệt:

```
Control plane (qua server):
  Browser → POST /api/live-token → Server → AI endpoint
                                          ← ephemeral token (30min TTL)
                                  ← token
Data plane (direct):
  Browser → WebSocket (ephemeral token) → AI
         ←←← Bidirectional audio streaming →→→
```

Kết quả:
- **Server không bao giờ chạm vào audio** → zero latency phía server cho data path
- **API key không bao giờ ra khỏi server** → client chỉ thấy token tạm thời

Đây là pattern tách **control plane** khỏi **data plane** — cùng cách Twilio, Daily.co, LiveKit thiết kế. → Chi tiết: [Bài 11 — Ephemeral Token & Security](./11-ephemeral-token-security.md)

---

## Pre-Connect — Giải quyết vấn đề số 5

Nhớ lại vấn đề thứ 5: 2–3 giây im lặng sau khi bắt máy. Pre-Connect Pattern giải quyết bằng cách **làm việc trong lúc điện thoại đang reo**.

```
KHÔNG có pre-connect:
  Bắt máy → [kết nối AI 800ms] → [load prompt 400ms] → [generate greeting 1.2s] → Nói
             └──────────────── 2-3 giây im lặng ────────────────┘

CÓ pre-connect:
  Đang reo → [kết nối AI] → [load prompt] → [cache greeting]   ← song song với tiếng reo
  Bắt máy  → Play cached greeting ← UNDER 100ms
```

**Không phải model nhanh hơn. Là làm sớm hơn.** → Chi tiết: [Bài 13 — Optimization Patterns](./13-optimization-patterns.md)

---

## 4 Kiến trúc AI Pipeline

Đây là quyết định kiến trúc quan trọng nhất — và không có "đúng/sai" tuyệt đối, chỉ có trade-off.

<div class="component-embed">

<!-- pipeline_architectures.html -->

<h2 class="sr-only">So sánh 4 kiến trúc AI pipeline trong Voice AI — Cascaded, Speech-Native, Thinker-Talker, LLM + Neural Codec</h2>
<style>
  .pl-wrap{font-family:system-ui,-apple-system,'Segoe UI',sans-serif;padding:.5rem 0;color:#e2e8f0}
  .pl-title{font-size:14px;font-weight:500;color:#e2e8f0;margin:0 0 12px}
  .pl-grid{display:grid;grid-template-columns:1fr 1fr;gap:8px;margin:0 0 10px}
  .pl-card{border-radius:10px;padding:14px 16px;border:.5px solid #2d3f55;background:#1e293b;display:flex;flex-direction:column;gap:10px}

  .pl-header{display:flex;align-items:flex-start;justify-content:space-between;gap:8px}
  .pl-name{font-size:13px;font-weight:600;color:#e2e8f0}
  .pl-badge{font-size:9px;font-weight:700;letter-spacing:.05em;padding:2px 7px;border-radius:20px;text-transform:uppercase;white-space:nowrap;flex-shrink:0}
  .badge-standard{background:rgba(59,130,246,.2);color:#60a5fa}
  .badge-low-lat{background:rgba(16,185,129,.2);color:#34d399}
  .badge-emerging{background:rgba(139,92,246,.2);color:#a78bfa}
  .badge-cheapest{background:rgba(245,158,11,.2);color:#fbbf24}

  .pl-metrics{display:grid;grid-template-columns:1fr 1fr;gap:6px}
  .pl-metric{background:#0f172a;border-radius:6px;padding:7px 9px}
  .pl-metric-label{font-size:10px;color:#64748b;margin:0 0 2px;letter-spacing:.03em}
  .pl-metric-val{font-size:13px;font-weight:600;font-family:ui-monospace,'Cascadia Code',monospace}
  .latency-slow{color:#fb923c}
  .latency-ok{color:#fbbf24}
  .latency-fast{color:#34d399}
  .cost-high{color:#94a3b8}
  .cost-mid{color:#94a3b8}
  .cost-low{color:#34d399;font-weight:700}

  .pl-flow{display:flex;align-items:center;gap:4px;flex-wrap:nowrap;overflow:hidden}
  .pl-flow-step{font-size:10px;font-weight:600;padding:3px 8px;border-radius:4px;white-space:nowrap;flex-shrink:0}
  .flow-blue{background:rgba(59,130,246,.2);color:#60a5fa}
  .flow-green{background:rgba(16,185,129,.2);color:#34d399}
  .flow-purple{background:rgba(139,92,246,.2);color:#a78bfa}
  .flow-orange{background:rgba(249,115,22,.2);color:#fb923c}
  .flow-yellow{background:rgba(245,158,11,.2);color:#fbbf24}
  .pl-flow-arr{font-size:11px;color:#475569;flex-shrink:0}
  .pl-flow-single{font-size:10px;font-weight:500;padding:3px 10px;border-radius:4px;flex:1;text-align:center}

  .pl-usecase{font-size:11px;color:#64748b;padding-top:6px;border-top:.5px solid #2d3f55;line-height:1.5}
  .pl-usecase-label{font-weight:600;color:#94a3b8}

  /* Comparison footer */
  .pl-footer{background:#1e293b;border-radius:8px;padding:12px 14px}
  .pl-footer-title{font-size:12px;font-weight:500;color:#e2e8f0;margin:0 0 10px}
  .pl-compare-grid{display:grid;grid-template-columns:90px repeat(4,1fr);gap:4px}
  .pl-col-header{font-size:10px;font-weight:700;color:#64748b;letter-spacing:.04em;text-align:center;padding:4px 2px}
  .pl-row-label{font-size:11px;color:#94a3b8;font-weight:500;padding:5px 0;display:flex;align-items:center}
  .pl-dots{text-align:center;font-size:12px;letter-spacing:1px;padding:5px 2px;line-height:1}
  .dot-row{border-top:.5px solid #1e2d3d}
  .dots-green{color:#34d399}
  .dots-yellow{color:#fbbf24}
  .dots-orange{color:#fb923c}
  .dots-gray{color:#334155}

  @media(max-width:520px){
    .pl-grid{grid-template-columns:1fr}
    .pl-compare-grid{grid-template-columns:70px repeat(4,1fr)}
    .pl-col-header{font-size:9px}
  }
</style>
<div class="pl-wrap">
  <div class="pl-title">4 kiến trúc AI pipeline — mỗi lựa chọn là trade-off</div>

  <div class="pl-grid">

    <!-- Cascaded -->
    <div class="pl-card">
      <div class="pl-header">
        <div class="pl-name">Cascaded Pipeline</div>
        <span class="pl-badge badge-standard">Production Standard</span>
      </div>
      <div class="pl-metrics">
        <div class="pl-metric">
          <div class="pl-metric-label">LATENCY</div>
          <div class="pl-metric-val latency-slow">600ms – 1.5s</div>
        </div>
        <div class="pl-metric">
          <div class="pl-metric-label">COST / MIN</div>
          <div class="pl-metric-val cost-high">$0.10 – 0.22</div>
        </div>
      </div>
      <div class="pl-flow">
        <span class="pl-flow-step flow-blue">STT</span>
        <span class="pl-flow-arr">→</span>
        <span class="pl-flow-step flow-purple">LLM</span>
        <span class="pl-flow-arr">→</span>
        <span class="pl-flow-step flow-green">TTS</span>
      </div>
      <div class="pl-usecase">
        <span class="pl-usecase-label">Phù hợp: </span>Healthcare · Legal · Enterprise — 128K+ context, audit trail
      </div>
    </div>

    <!-- Speech-Native -->
    <div class="pl-card">
      <div class="pl-header">
        <div class="pl-name">Speech-Native</div>
        <span class="pl-badge badge-low-lat">Lowest Latency</span>
      </div>
      <div class="pl-metrics">
        <div class="pl-metric">
          <div class="pl-metric-label">LATENCY</div>
          <div class="pl-metric-val latency-fast">160 – 400ms</div>
        </div>
        <div class="pl-metric">
          <div class="pl-metric-label">COST / MIN</div>
          <div class="pl-metric-val cost-mid">$0.06 – 0.15</div>
        </div>
      </div>
      <div class="pl-flow">
        <span class="pl-flow-step flow-green pl-flow-single">Single Model: Listen + Reason + Speak</span>
      </div>
      <div class="pl-usecase">
        <span class="pl-usecase-label">Phù hợp: </span>Consumer apps · UX-first — GPT-4o Realtime, Gemini Live
      </div>
    </div>

    <!-- Thinker-Talker -->
    <div class="pl-card">
      <div class="pl-header">
        <div class="pl-name">Thinker-Talker</div>
        <span class="pl-badge badge-emerging">Emerging 2025</span>
      </div>
      <div class="pl-metrics">
        <div class="pl-metric">
          <div class="pl-metric-label">LATENCY</div>
          <div class="pl-metric-val latency-fast">~257ms</div>
        </div>
        <div class="pl-metric">
          <div class="pl-metric-label">COST / MIN</div>
          <div class="pl-metric-val cost-mid">$0.08 – 0.18</div>
        </div>
      </div>
      <div class="pl-flow">
        <span class="pl-flow-step flow-purple">Thinker</span>
        <span class="pl-flow-arr">→</span>
        <span class="pl-flow-step flow-orange">Talker</span>
      </div>
      <div class="pl-usecase">
        <span class="pl-usecase-label">Phù hợp: </span>New projects 2025+ — full LLM ecosystem, latency thấp hơn cascaded
      </div>
    </div>

    <!-- LLM + Neural Codec -->
    <div class="pl-card">
      <div class="pl-header">
        <div class="pl-name">LLM + Neural Codec</div>
        <span class="pl-badge badge-cheapest">Cheapest</span>
      </div>
      <div class="pl-metrics">
        <div class="pl-metric">
          <div class="pl-metric-label">LATENCY</div>
          <div class="pl-metric-val latency-fast">&lt; 50ms stream</div>
        </div>
        <div class="pl-metric">
          <div class="pl-metric-label">COST / MIN</div>
          <div class="pl-metric-val cost-low">&lt; $0.012</div>
        </div>
      </div>
      <div class="pl-flow">
        <span class="pl-flow-step flow-purple">LLM</span>
        <span class="pl-flow-arr">→</span>
        <span class="pl-flow-step flow-yellow">Neural Codec</span>
      </div>
      <div class="pl-usecase">
        <span class="pl-usecase-label">Phù hợp: </span>High-volume · Self-hosted — Orpheus, Sesame CSM
      </div>
    </div>

  </div>

  <!-- Comparison matrix -->
  <div class="pl-footer">
    <div class="pl-footer-title">So sánh nhanh</div>
    <div class="pl-compare-grid">
      <div></div>
      <div class="pl-col-header">Cascaded</div>
      <div class="pl-col-header">Speech-Native</div>
      <div class="pl-col-header">Thinker-Talker</div>
      <div class="pl-col-header">Neural Codec</div>

      <div class="pl-row-label dot-row">Latency</div>
      <div class="pl-dots dot-row dots-orange">●●○○</div>
      <div class="pl-dots dot-row dots-green">●●●●</div>
      <div class="pl-dots dot-row dots-green">●●●○</div>
      <div class="pl-dots dot-row dots-green">●●●●</div>

      <div class="pl-row-label dot-row">Control</div>
      <div class="pl-dots dot-row dots-green">●●●●</div>
      <div class="pl-dots dot-row dots-orange">●●○○</div>
      <div class="pl-dots dot-row dots-green">●●●○</div>
      <div class="pl-dots dot-row dots-yellow">●●○○</div>

      <div class="pl-row-label dot-row">Cost</div>
      <div class="pl-dots dot-row dots-yellow">●●○○</div>
      <div class="pl-dots dot-row dots-yellow">●●●○</div>
      <div class="pl-dots dot-row dots-yellow">●●○○</div>
      <div class="pl-dots dot-row dots-green">●●●●</div>

      <div class="pl-row-label dot-row">Maturity</div>
      <div class="pl-dots dot-row dots-green">●●●●</div>
      <div class="pl-dots dot-row dots-green">●●●○</div>
      <div class="pl-dots dot-row dots-yellow">●●○○</div>
      <div class="pl-dots dot-row dots-orange">●○○○</div>
    </div>
  </div>
</div>

</div>

→ So sánh chi tiết: [Bài 07 — 4 AI Pipeline Architectures](./07-ai-pipeline-architectures.md)

---

## Các Pattern Kiến trúc Production

### Unified Server — Anti-microservices có chủ đích

Gom HTTP + WebSocket + audio bridge vào cùng 1 process. Nghe "sai" theo convention microservices — nhưng đúng cho Voice AI.

**Lý do:** Audio bridge truy cập session state mỗi ~20ms. Nếu tách microservices, mỗi operation cần 1 network round-trip → 20ms/hop × nhiều hops = latency budget vỡ ngay. In-memory access là ~0.1µs. Network access là ~0.5ms. **Khác nhau 5.000 lần.**

### Speculative Tool Calling — Che latency bằng parallelism

```
User đang nói: "Tôi muốn đặt vé máy bay từ Hà Nội..."
                                                        ↑
                              AI đã bắt đầu gọi flight API ở đây
User nói xong: "...đến TP.HCM ngày mai"
AI respond: "Tôi thấy có 3 chuyến..." ← data đã có sẵn, near-zero tool latency
```

### Barge-in State Machine — User có quyền ngắt lời

```
States: LISTENING → PROCESSING → SPEAKING
                                    ↑
                          Nếu VAD detect user nói:
                          → Cancel TTS stream ngay lập tức
                          → Transition về LISTENING
                          → Không để AI "nói đè"
```

> ⚠️ **Lỗi phổ biến:** Xử lý barge-in bằng cách chờ AI nói xong rồi mới process interrupt. User đã cúp máy lâu rồi.

### Template-Driven System

Không hardcode conversation flow trong code. Persona, prompt, flow config đều là dữ liệu trong Template Store — thay đổi behavior của AI mà không cần deploy lại code.

→ Chi tiết: [Bài 12 — Unified Server](./12-unified-server-pattern.md) · [Bài 13 — Optimization Patterns](./13-optimization-patterns.md)

---

## Tóm tắt — Bức tranh tổng thể

Nhìn lại toàn bộ bài viết: Audio Bridge, Ephemeral Token, Pre-Connect, Barge-in State Machine, Unified Server — **không cái nào liên quan đến model AI**. Tất cả đều là system engineering thuần túy.

Đây là lý do 90% dự án Voice AI chết ở production: team build demo xong nghĩ "chỉ cần thêm Twilio là xong". Thực tế, Twilio chỉ là 1 dòng trong danh sách 20 thành phần cần xử lý đúng.

**Series này giải quyết 20 thành phần đó — từng bài một, từng gốc lên.**

---

## Đọc tiếp — Chọn path phù hợp với bạn

**Bạn mới với Voice AI** → Bắt đầu tuần tự từ Tầng 1:

→ **[Bài 01 — Audio Fundamentals](./01-audio-fundamentals.md)**
Hiểu PCM, sample rate, G.711, Opus — nền tảng để đọc mọi thứ còn lại

---

**Bạn đang debug production** → Đọc 3 bài này trước:

→ [Bài 04 — Audio Bridge](./04-audio-bridge.md) · [Bài 06 — Latency Budget](./06-latency-budget.md) · [Bài 14 — Observability](./14-observability.md)

---

**Bạn đang chọn kiến trúc cho dự án mới** → 2 bài quyết định:

→ [Bài 07 — 4 AI Pipeline Architectures](./07-ai-pipeline-architectures.md) · [Bài 15 — Cost Analysis](./15-cost-analysis.md)

---

**Toàn bộ roadmap 15 bài** → [Voice AI System Architecture — README](./README.md)

---

*Bài 1/15 · [Bài 01 — Audio Fundamentals →](./01-audio-fundamentals.md) · [Roadmap](./README.md)*
