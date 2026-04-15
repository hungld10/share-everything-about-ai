# Bài 01 — Audio Fundamentals cho Voice AI

> **Tầng 1 — Nền tảng** | Series: [Voice AI System Architecture](./README.md)
> ⏱ ~10 phút đọc · 💡 Sau bài này: hiểu tại sao Voice AI phone channel phức tạp hơn browser, và tại sao mỗi bước trong pipeline đều có "giá" latency

---

Tại sao giọng AI khi gọi điện thoại luôn nghe "mỏng" và khác hẳn khi dùng web app — dù cùng một model?

Câu trả lời không phải ở model. Không phải ở prompt. Mà ở một con số: **8.000**.

Đó là số lần mỗi giây mà hệ thống điện thoại "chụp ảnh" âm thanh giọng nói — một chuẩn được đặt ra từ năm 1972 và vẫn chưa thay đổi. Web app dùng 48.000 lần/giây. Sự khác biệt đó — 8kHz vs 48kHz — là nguồn gốc của toàn bộ bài toán Audio Bridge mà bài 04 sẽ đi sâu.

Nhưng trước tiên: audio số là gì, và tại sao con số sample rate lại quan trọng đến vậy?

---

## Âm thanh số — từ sóng vật lý đến dãy số

Âm thanh trong tự nhiên là sóng áp suất không khí — liên tục, analog. Máy tính không thể lưu sóng liên tục, nên phải "chụp ảnh" sóng đó nhiều lần mỗi giây, mỗi lần ghi lại một con số đại diện cho biên độ (amplitude) tại thời điểm đó. Quá trình này gọi là **sampling** (lấy mẫu), và dãy số thu được chính là **audio số** (digital audio).

![Sampling waveform — PCM pulse code modulation. Sóng analog được lấy mẫu 16.000 lần/giây tại 16kHz](./images/sampling-waveform.png)

Hai thông số quyết định chất lượng audio số:

**Sample rate** (tần số lấy mẫu) — bao nhiêu lần "chụp" mỗi giây, đo bằng Hz hoặc kHz. Theo định lý Nyquist, sample rate phải gấp đôi tần số cao nhất muốn tái tạo. Tai người nghe được đến ~20kHz nên CD dùng 44.1kHz. Nhưng giọng nói chỉ nằm trong khoảng 300Hz–3.4kHz (điện thoại) hoặc đến ~8kHz (wideband) — không cần sample rate cao như nhạc.

**Bit depth** (độ sâu bit) — mỗi lần chụp, dùng bao nhiêu bit để ghi con số biên độ. 8-bit cho 256 mức, 16-bit cho 65.536 mức. Bit depth cao hơn = chi tiết hơn = ít nhiễu lượng tử hóa hơn (quantization noise). Voice AI thường dùng 16-bit — đủ chi tiết cho giọng nói mà không quá nặng.

Kết hợp hai thông số này tạo ra **PCM** — khái niệm nền tảng nhất trong toàn bộ series.

---

## PCM — "ngôn ngữ chung" của Voice AI pipeline

**PCM (Pulse Code Modulation)** là cách biểu diễn audio số đơn giản nhất: một dãy số tuần tự, mỗi số là biên độ tại một thời điểm, không nén, không biến đổi. Mọi codec audio cuối cùng đều phải decode về PCM trước khi xử lý hoặc phát.

PCM quan trọng trong Voice AI vì đây là ngôn ngữ mà AI model "hiểu": model nhận **PCM 16kHz** đầu vào và xuất **PCM 24kHz** đầu ra. G.711 từ điện thoại phải decode về PCM. Opus từ browser phải decode về PCM. Sau đó encode ngược lại để gửi đi.

**PCM nặng như thế nào?**
1 giây audio PCM 16kHz, 16-bit = 16.000 samples × 2 bytes = **32KB**.
Một cuộc gọi 5 phút = **~9.6MB** chỉ riêng audio thô — chưa kể metadata hay protocol overhead.

Đây là lý do cần codec để truyền qua mạng. Nhưng mọi codec đều phải "về PCM" trước khi AI xử lý — và đó chính là nơi latency tích lũy.

> ⚠️ **Misconception phổ biến:** *"Dùng PCM 48kHz cho AI model cho chất lượng cao hơn."* — Thực ra model chỉ cần 16kHz đầu vào. Nếu feed 48kHz, model tự downsample — bạn vừa tốn bandwidth, vừa tốn CPU, mà không được gì thêm.

---

## 4 sample rate cần thuộc lòng

Đây là phần nhiều người bị lẫn lộn nhất — vì mỗi thành phần trong pipeline dùng một con số khác nhau và đều có lý do kỹ thuật cụ thể.

**8kHz — Thế giới điện thoại**
Chuẩn PSTN từ thập niên 1960. Chỉ tái tạo được tần số đến 4kHz, đủ cho giọng nói nhưng nghe "mỏng" và thiếu chi tiết. Khi user gọi từ điện thoại thật, audio *luôn luôn* đến server ở 8kHz — vì hạ tầng viễn thông toàn cầu đặt chuẩn này từ 50 năm trước và chưa thay đổi.

**16kHz — Thế giới AI input**
Chuẩn mà hầu hết STT engine và voice model nhận input. Tái tạo được đến 8kHz, đủ để model nhận diện giọng nói với độ chính xác cao. Đây là lý do pipeline inbound phải upsample 8→16kHz trước khi đưa vào model.

**24kHz — Thế giới AI output**
Chuẩn output của TTS engine và voice model hiện đại. Cho chất lượng giọng nói tự nhiên hơn 16kHz rõ rệt — nghe "ấm" và có chiều sâu hơn. Pipeline outbound phải downsample 24→8kHz khi gửi về điện thoại.

**48kHz — Thế giới browser**
Chuẩn WebRTC. Khi user dùng web app, audio được thu và phát ở 48kHz với codec Opus — chất lượng cao nhất trong toàn bộ pipeline. Channel browser *không bắt buộc* phải qua server, có thể stream thẳng browser ↔ AI.

<div class="component-embed">

<!-- sample_rate_blocks.html -->

<h2 class="sr-only">4 sample rate quan trọng trong Voice AI pipeline — 8kHz, 16kHz, 24kHz, 48kHz</h2>
<style>
  .sr-wrap{font-family:system-ui,-apple-system,'Segoe UI',sans-serif;padding:.5rem 0;color:#e2e8f0}
  .sr-title{font-size:14px;font-weight:500;color:#e2e8f0;margin:0 0 12px}
  .sr-blocks{display:grid;grid-template-columns:repeat(4,1fr);gap:8px;margin:0 0 10px}
  .sr-block{border-radius:10px;padding:14px 12px;border:.5px solid #2d3f55;background:#1e293b;display:flex;flex-direction:column;gap:8px;position:relative;overflow:hidden}

  /* Colored top accent bar */
  .sr-block::before{content:'';position:absolute;top:0;left:0;right:0;height:3px}
  .sr-block-8::before{background:#6b7280}
  .sr-block-16::before{background:#3b82f6}
  .sr-block-24::before{background:#10b981}
  .sr-block-48::before{background:#8b5cf6}

  .sr-icon{font-size:18px;line-height:1}
  .sr-label{font-size:10px;font-weight:700;letter-spacing:.07em;text-transform:uppercase}
  .sr-label-8{color:#9ca3af}
  .sr-label-16{color:#60a5fa}
  .sr-label-24{color:#34d399}
  .sr-label-48{color:#a78bfa}

  .sr-freq{font-size:22px;font-weight:700;font-family:ui-monospace,'Cascadia Code',monospace;line-height:1.1}
  .sr-freq-8{color:#9ca3af}
  .sr-freq-16{color:#60a5fa}
  .sr-freq-24{color:#34d399}
  .sr-freq-48{color:#a78bfa}
  .sr-unit{font-size:12px;font-weight:400}

  .sr-note{font-size:11px;color:#94a3b8;line-height:1.4}
  .sr-context{font-size:10px;color:#475569;font-family:ui-monospace,'Cascadia Code',monospace}

  .sr-codec{display:inline-block;font-size:9px;font-weight:700;padding:2px 6px;border-radius:3px;letter-spacing:.03em}
  .codec-8{background:rgba(107,114,128,.2);color:#9ca3af}
  .codec-16{background:rgba(59,130,246,.2);color:#60a5fa}
  .codec-24{background:rgba(16,185,129,.2);color:#34d399}
  .codec-48{background:rgba(139,92,246,.2);color:#a78bfa}

  .sr-quality{margin:2px 0 0}
  .sr-quality-label{font-size:10px;color:#475569;margin:0 0 4px}
  .sr-quality-track{height:4px;border-radius:2px;background:#0f172a;overflow:hidden}
  .sr-quality-fill{height:100%;border-radius:2px}
  .fill-8{width:25%;background:#6b7280}
  .fill-16{width:50%;background:#3b82f6}
  .fill-24{width:75%;background:#10b981}
  .fill-48{width:100%;background:linear-gradient(90deg,#8b5cf6,#a78bfa)}

  /* Conversion bridge */
  .sr-bridge-wrap{background:#1e293b;border-radius:8px;padding:12px 14px;margin:0 0 8px}
  .sr-bridge-title{font-size:12px;font-weight:500;color:#e2e8f0;margin:0 0 10px}
  .sr-bridge{display:flex;align-items:center;gap:4px;overflow-x:auto}
  .sr-bridge-step{display:flex;flex-direction:column;align-items:center;gap:3px;flex:1;min-width:50px}
  .sr-bridge-khz{font-size:13px;font-weight:700;font-family:ui-monospace,'Cascadia Code',monospace}
  .sr-bridge-who{font-size:9px;color:#64748b;text-align:center;line-height:1.3}
  .sr-bridge-arr{font-size:13px;color:#334155;flex-shrink:0}
  .sr-bridge-op{display:flex;flex-direction:column;align-items:center;flex-shrink:0}
  .sr-bridge-op-label{font-size:9px;font-weight:700;padding:2px 6px;border-radius:3px}
  .op-up{background:rgba(59,130,246,.2);color:#60a5fa}
  .op-down{background:rgba(249,115,22,.2);color:#fb923c}

  .note-box{background:rgba(59,130,246,.1);border-radius:8px;padding:10px 14px;font-size:12px;color:#93c5fd;line-height:1.5}

  @media(max-width:520px){
    .sr-blocks{grid-template-columns:repeat(2,1fr)}
    .sr-freq{font-size:18px}
  }
</style>
<div class="sr-wrap">
  <div class="sr-title">4 sample rate trong Voice AI — mỗi con số đều có lý do kỹ thuật</div>

  <div class="sr-blocks">
    <div class="sr-block sr-block-8">
      <div class="sr-icon">📞</div>
      <div>
        <div class="sr-label sr-label-8">PSTN · Phone</div>
        <div class="sr-freq sr-freq-8">8<span class="sr-unit"> kHz</span></div>
      </div>
      <div class="sr-note">Narrowband — chuẩn điện thoại từ 1972</div>
      <span class="sr-codec codec-8">G.711 µ-law</span>
      <div class="sr-quality">
        <div class="sr-quality-label">Quality</div>
        <div class="sr-quality-track"><div class="sr-quality-fill fill-8"></div></div>
      </div>
      <div class="sr-context">4 kHz max freq</div>
    </div>

    <div class="sr-block sr-block-16">
      <div class="sr-icon">🧠</div>
      <div>
        <div class="sr-label sr-label-16">AI Model · Input</div>
        <div class="sr-freq sr-freq-16">16<span class="sr-unit"> kHz</span></div>
      </div>
      <div class="sr-note">Wideband — chuẩn STT &amp; voice model</div>
      <span class="sr-codec codec-16">PCM 16-bit</span>
      <div class="sr-quality">
        <div class="sr-quality-label">Quality</div>
        <div class="sr-quality-track"><div class="sr-quality-fill fill-16"></div></div>
      </div>
      <div class="sr-context">8 kHz max freq</div>
    </div>

    <div class="sr-block sr-block-24">
      <div class="sr-icon">🔊</div>
      <div>
        <div class="sr-label sr-label-24">AI Model · Output</div>
        <div class="sr-freq sr-freq-24">24<span class="sr-unit"> kHz</span></div>
      </div>
      <div class="sr-note">Chuẩn TTS hiện đại — giọng tự nhiên</div>
      <span class="sr-codec codec-24">PCM 16-bit</span>
      <div class="sr-quality">
        <div class="sr-quality-label">Quality</div>
        <div class="sr-quality-track"><div class="sr-quality-fill fill-24"></div></div>
      </div>
      <div class="sr-context">12 kHz max freq</div>
    </div>

    <div class="sr-block sr-block-48">
      <div class="sr-icon">🌐</div>
      <div>
        <div class="sr-label sr-label-48">Browser · WebRTC</div>
        <div class="sr-freq sr-freq-48">48<span class="sr-unit"> kHz</span></div>
      </div>
      <div class="sr-note">Chất lượng cao nhất trong pipeline</div>
      <span class="sr-codec codec-48">Opus (RFC 6716)</span>
      <div class="sr-quality">
        <div class="sr-quality-label">Quality</div>
        <div class="sr-quality-track"><div class="sr-quality-fill fill-48"></div></div>
      </div>
      <div class="sr-context">24 kHz max freq</div>
    </div>
  </div>

  <div class="sr-bridge-wrap">
    <div class="sr-bridge-title">Luồng chuyển đổi khi Phone gọi vào hệ thống</div>
    <div class="sr-bridge">
      <div class="sr-bridge-step">
        <span class="sr-bridge-khz sr-freq-8">8kHz</span>
        <span class="sr-bridge-who">Phone<br>G.711</span>
      </div>
      <span class="sr-bridge-arr">→</span>
      <div class="sr-bridge-op"><span class="sr-bridge-op-label op-up">↑ upsample</span></div>
      <span class="sr-bridge-arr">→</span>
      <div class="sr-bridge-step">
        <span class="sr-bridge-khz sr-freq-16">16kHz</span>
        <span class="sr-bridge-who">AI input<br>PCM</span>
      </div>
      <span class="sr-bridge-arr">→</span>
      <div class="sr-bridge-step">
        <span class="sr-bridge-khz sr-freq-24">24kHz</span>
        <span class="sr-bridge-who">AI output<br>PCM</span>
      </div>
      <span class="sr-bridge-arr">→</span>
      <div class="sr-bridge-op"><span class="sr-bridge-op-label op-down">↓ downsample</span></div>
      <span class="sr-bridge-arr">→</span>
      <div class="sr-bridge-step">
        <span class="sr-bridge-khz sr-freq-8">8kHz</span>
        <span class="sr-bridge-who">Phone<br>G.711</span>
      </div>
    </div>
  </div>

  <div class="note-box">Sự không khớp giữa 4 con số này là toàn bộ lý do tồn tại của Audio Bridge. Browser channel (48kHz) có thể bypass conversion bằng direct WebSocket — Phone channel (8kHz) thì không.</div>
</div>

</div>

Chính sự không khớp giữa 4 con số này — đặc biệt là 8kHz của phone vs 16/24kHz của AI — là toàn bộ lý do tồn tại của Audio Bridge. Sẽ đi sâu ở [Bài 04](./04-audio-bridge.md).

---

## G.711 — codec sinh ra từ năm 1972 vẫn đang chạy hôm nay

**G.711** là codec audio tiêu chuẩn của mạng điện thoại toàn cầu (PSTN), được ITU chuẩn hóa năm 1972. Nghe có vẻ cổ xưa — nhưng đây là codec mà *mọi cuộc gọi điện thoại trên thế giới* vẫn đang sử dụng, kể cả cuộc gọi qua Twilio hay Telnyx vào hệ thống Voice AI của bạn.

**Tại sao G.711 vẫn tồn tại?** Vì có hàng tỷ thiết bị và hàng triệu km cáp viễn thông được thiết kế xung quanh chuẩn này. Thay thế G.711 nghĩa là thay thế toàn bộ hạ tầng viễn thông toàn cầu — điều không ai làm. Đây là ví dụ điển hình của "legacy lock-in" quy mô hành tinh.

G.711 dùng kỹ thuật **companding** (nén-giãn) — thay vì ghi biên độ tuyến tính (linear PCM), nó dùng thang logarithmic. Âm thanh nhỏ được ghi với độ phân giải cao hơn, âm thanh lớn bị nén lại. Điều này tận dụng đặc tính tai người: nhạy hơn với thay đổi ở âm lượng nhỏ hơn là ở âm lượng lớn.

Có 2 biến thể: **µ-law** (Bắc Mỹ, Nhật) và **A-law** (châu Âu, phần còn lại). Cả hai đều nén 14-bit linear PCM xuống 8-bit logarithmic → bitrate cố định **64kbps**.

**Hệ quả cho Voice AI:** Khi user gọi qua số điện thoại thật, audio đến server ở dạng G.711 µ-law 8kHz, 8-bit. Server phải:
```
G.711 µ-law → Decode → Linear PCM 8kHz → Upsample → PCM 16kHz → AI Model
```
Và chiều ngược lại:
```
AI Model PCM 24kHz → Downsample → PCM 8kHz → Encode G.711 → Điện thoại
```

**Mỗi bước chuyển đổi đều có giá — latency tăng và chất lượng có thể giảm nếu làm sai.**

---

## Opus — codec được thiết kế cho internet

**Opus** là codec audio hiện đại, chuẩn mở (RFC 6716, 2012), được thiết kế đặc biệt cho truyền thông realtime qua internet. Đây là codec mặc định của WebRTC — nghĩa là khi user dùng browser để nói chuyện với AI, audio được encode bằng Opus.

Opus kết hợp 2 công nghệ: **SILK** (phát triển bởi Skype, tối ưu cho giọng nói) và **CELT** (tối ưu cho âm nhạc/âm thanh tổng quát). Nó tự động chuyển đổi giữa 2 mode tùy nội dung.

So sánh Opus với G.711:

Về bitrate, Opus hỗ trợ 6kbps đến 510kbps, tự điều chỉnh theo băng thông mạng. G.711 cố định 64kbps. Ở 24kbps, Opus đã cho chất lượng giọng nói tốt hơn G.711 ở 64kbps.

Về sample rate, Opus hỗ trợ 8/12/16/24/48kHz, trong khi G.711 chỉ 8kHz.

Về độ trễ, Opus có thể xuống 2.5ms frame size (thường dùng 20ms), G.711 xử lý từng sample nên gần như zero algorithmic delay.

Về khả năng thích ứng, Opus có FEC (Forward Error Correction) tích hợp — tự chèn dữ liệu dự phòng để chống mất gói tin. G.711 không có — mất gói = mất audio.

Tại sao Opus quan trọng trong Voice AI: channel browser (WebRTC) dùng Opus 48kHz, cho chất lượng audio cao nhất có thể. Khi server nhận audio từ browser, nó cần decode Opus → PCM → có thể resample về 16kHz cho model. Nhưng vì Opus qua WebRTC thường được thiết lập direct connection (browser ↔ AI model), server có thể không cần chạm vào audio — đây là khác biệt kiến trúc cơ bản giữa channel browser và phone.

---

## Tại sao tất cả điều này quan trọng cho Voice AI

Có một engineer đã deploy Voice AI lên production. Tiếng nghe méo, AI nhận diện sai tên người dùng. Anh đổi model 3 lần. Thêm noise cancellation. Tinh chỉnh prompt. Vẫn méo.

Nguyên nhân thật: code audio bridge upsample 8kHz → 16kHz dùng **nearest-neighbor interpolation** thay vì **sinc interpolation** — sai về mặt tín hiệu số, tạo ra aliasing artifact. Không biết sample rate và cách resampling hoạt động, không bao giờ tìm ra lỗi này. Model không giúp gì được.

Audio fundamentals là nền tảng để tránh class of bugs này:

- Không hiểu **sample rate** → không hiểu tại sao cần upsample/downsample, không biết chọn algorithm nào
- Không hiểu **G.711** → không hiểu tại sao channel phone phức tạp hơn browser
- Không hiểu **PCM** → không hiểu "ngôn ngữ chung" giữa các thành phần
- Không hiểu **codec overhead** → không debug được latency budget — mỗi bước encode/decode đều tốn thời gian

<div class="component-embed">

<!-- audio_fundamentals_codec_comparison.html -->
<h2 class="sr-only">So sánh thông số kỹ thuật giữa các codec và sample rate trong Voice AI pipeline</h2>
<style>
  .wrap{font-family:system-ui,-apple-system,'Segoe UI',sans-serif;padding:.5rem 0;color:#e2e8f0}
  .section{margin:0 0 1.5rem}
  .section-title{font-size:14px;font-weight:500;color:#e2e8f0;margin:0 0 10px}
  .row{display:flex;gap:8px;margin:0 0 8px}
  .col{flex:1;background:#1e293b;border-radius:8px;padding:12px 14px}
  .col-label{font-size:11px;color:#64748b;margin:0 0 4px;letter-spacing:.03em;text-transform:uppercase;font-weight:600;font-size:10px}
  .col-val{font-size:16px;font-weight:700;color:#e2e8f0;margin:0;font-family:ui-monospace,'Cascadia Code',monospace}
  .col-note{font-size:11px;color:#94a3b8;margin:4px 0 0;line-height:1.4}
  .vs{display:grid;grid-template-columns:1fr 1fr;gap:8px;margin:0 0 8px}
  .vs-card{border-radius:10px;padding:14px 16px;border:.5px solid #334155;background:#1e293b}
  .vs-h{font-size:13px;font-weight:600;color:#e2e8f0;margin:0 0 10px}
  .vs-row{display:flex;justify-content:space-between;font-size:12px;padding:5px 0;border-top:.5px solid #2d3f55}
  .vs-row:first-of-type{border-top:none}
  .vs-k{color:#94a3b8}
  .vs-v{color:#e2e8f0;font-weight:600;font-family:ui-monospace,'Cascadia Code',monospace;font-size:11px}
  .pipeline{display:flex;align-items:center;gap:0;margin:8px 0}
  .pipe-step{flex:1;text-align:center;padding:10px 6px;font-size:11px;color:#94a3b8;line-height:1.4;border-radius:8px;background:#1e293b}
  .pipe-step strong{display:block;font-size:12px;color:#e2e8f0;font-weight:600}
  .pipe-arr{font-size:14px;color:#475569;padding:0 3px;flex-shrink:0}
  .note-box{background:rgba(59,130,246,.1);border-radius:8px;padding:10px 14px;font-size:12px;color:#93c5fd;line-height:1.5;margin:10px 0 0}
  @media(max-width:480px){.vs{grid-template-columns:1fr}.row{flex-direction:column}}
</style>
<div class="wrap">

<div class="section">
  <div class="section-title">Sample rate trong Voice AI pipeline</div>
  <div class="row">
    <div class="col">
      <div class="col-label">PSTN / Điện thoại</div>
      <div class="col-val">8 kHz</div>
      <div class="col-note">G.711, narrowband, chuẩn từ 1972</div>
    </div>
    <div class="col">
      <div class="col-label">AI model input</div>
      <div class="col-val">16 kHz</div>
      <div class="col-note">Wideband, chuẩn cho hầu hết STT/voice model</div>
    </div>
    <div class="col">
      <div class="col-label">AI model output</div>
      <div class="col-val">24 kHz</div>
      <div class="col-note">Giọng nói tự nhiên, chuẩn TTS hiện đại</div>
    </div>
    <div class="col">
      <div class="col-label">Browser / WebRTC</div>
      <div class="col-val">48 kHz</div>
      <div class="col-note">Opus codec, chất lượng cao nhất</div>
    </div>
  </div>
</div>

<div class="section">
  <div class="section-title">G.711 vs Opus</div>
  <div class="vs">
    <div class="vs-card">
      <div class="vs-h">G.711 µ-law</div>
      <div class="vs-row"><span class="vs-k">Ra đời</span><span class="vs-v">1972</span></div>
      <div class="vs-row"><span class="vs-k">Dùng cho</span><span class="vs-v">PSTN / Phone</span></div>
      <div class="vs-row"><span class="vs-k">Sample rate</span><span class="vs-v">8 kHz</span></div>
      <div class="vs-row"><span class="vs-k">Bitrate</span><span class="vs-v">64 kbps cố định</span></div>
      <div class="vs-row"><span class="vs-k">Nén</span><span class="vs-v">Logarithmic (µ-law)</span></div>
      <div class="vs-row"><span class="vs-k">Chống mất gói</span><span class="vs-v">Không</span></div>
      <div class="vs-row"><span class="vs-k">Algorithmic delay</span><span class="vs-v">~0ms</span></div>
    </div>
    <div class="vs-card">
      <div class="vs-h">Opus (RFC 6716)</div>
      <div class="vs-row"><span class="vs-k">Ra đời</span><span class="vs-v">2012</span></div>
      <div class="vs-row"><span class="vs-k">Dùng cho</span><span class="vs-v">WebRTC / Browser</span></div>
      <div class="vs-row"><span class="vs-k">Sample rate</span><span class="vs-v">8–48 kHz</span></div>
      <div class="vs-row"><span class="vs-k">Bitrate</span><span class="vs-v">6–510 kbps adaptive</span></div>
      <div class="vs-row"><span class="vs-k">Nén</span><span class="vs-v">SILK + CELT hybrid</span></div>
      <div class="vs-row"><span class="vs-k">Chống mất gói</span><span class="vs-v">FEC tích hợp</span></div>
      <div class="vs-row"><span class="vs-k">Algorithmic delay</span><span class="vs-v">2.5–20ms</span></div>
    </div>
  </div>
</div>

<div class="section">
  <div class="section-title">Luồng chuyển đổi: Phone → AI → Phone</div>
  <div class="pipeline">
    <div class="pipe-step"><strong>Phone</strong>G.711 µ-law<br>8kHz, 8-bit</div>
    <span class="pipe-arr">→</span>
    <div class="pipe-step"><strong>Decode</strong>G.711 →<br>Linear PCM</div>
    <span class="pipe-arr">→</span>
    <div class="pipe-step"><strong>Upsample</strong>8kHz →<br>16kHz</div>
    <span class="pipe-arr">→</span>
    <div class="pipe-step"><strong>AI Model</strong>PCM 16kHz in<br>24kHz out</div>
    <span class="pipe-arr">→</span>
    <div class="pipe-step"><strong>Downsample</strong>24kHz →<br>8kHz</div>
    <span class="pipe-arr">→</span>
    <div class="pipe-step"><strong>Encode</strong>Linear PCM →<br>G.711</div>
    <span class="pipe-arr">→</span>
    <div class="pipe-step"><strong>Phone</strong>G.711 µ-law<br>8kHz</div>
  </div>
  <div class="note-box">Mỗi bước chuyển đổi đều tốn latency và có thể mất chất lượng. Đây là lý do Audio Bridge (bài 04) là thành phần phức tạp nhất của Voice AI production.</div>
</div>

</div>

</div>

---

## Tóm tắt — những gì cần nhớ

**PCM** là "ngôn ngữ chung" — mọi audio trong pipeline đều phải decode về PCM trước khi AI xử lý. Mọi bước encode/decode đều có giá latency.

**4 sample rate:** 8kHz (phone) → 16kHz (AI input) → 24kHz (AI output) → 48kHz (browser). Sự không khớp giữa chúng là toàn bộ lý do tồn tại của Audio Bridge.

**G.711 là thực tế không thể thay đổi** — hạ tầng viễn thông toàn cầu sẽ không upgrade trong tương lai gần. Hệ thống Voice AI production phải nói được "ngôn ngữ 1972" này.

**Opus và G.711 đại diện cho 2 thế giới** — internet adaptive vs PSTN fixed. Voice AI production phải bridge cả hai.

---

## Bài tiếp theo

Bài 01 giải thích *cái gì* đang được truyền — dãy số PCM, đóng gói bởi G.711 hoặc Opus.

**Bài 02 — WebSocket & Realtime Communication** (sắp ra) giải thích *truyền bằng cách nào*. Cụ thể: tại sao HTTP request/response không dùng được cho audio streaming, WebSocket lifecycle hoạt động ra sao, và một khái niệm ít được nhắc đến nhưng gây nhiều bug production: **backpressure** — điều gì xảy ra khi AI sinh audio nhanh hơn tốc độ network có thể gửi đi?

---

*Bài 1/15 · ← [Tổng quan](./00-overview.md) · [Roadmap](./README.md)*
