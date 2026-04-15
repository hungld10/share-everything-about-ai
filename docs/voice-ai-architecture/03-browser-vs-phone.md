# Bài 03 — Browser vs Phone: 2 Thế Giới Kết Nối

> **Tầng 1 — Nền tảng** | Series: [Voice AI System Architecture](./README.md)
> ⏱ ~14 phút đọc · 💡 Sau bài này: hiểu tại sao browser và phone tạo ra hai "thế giới" hoàn toàn khác nhau, và tại sao cùng một AI Model phải có hai pipeline xử lý khác biệt

---

## Bài toán cốt lõi — Một AI Model, hai channel hoàn toàn khác nhau

Nếu bạn xây dựng Voice AI system, khả năng cao sẽ có 2 hướng kết nối:

1. **Browser/Web App** — User nói vào microphone → WebRTC → AI → TTS → Speaker
2. **Điện thoại thực** — User gọi số điện thoại → PSTN → Twilio WebSocket → AI → TTS → lại điện thoại

Cả hai đều phải đến cùng một AI Model. Nhưng:
- Browser có **Opus 48kHz** tích hợp sẵn
- Phone có **G.711 8kHz** từ Twilio

Đây không phải chỉ là codec khác nhau. Đó là **hai thế giới** về audio quality, latency, kiến trúc xử lý, cách quản lý kết nối.

---

## Thế giới 1: Browser — WebRTC, Opus, Full Quality

### Audio Codec: Opus ở 48 kHz

Browser Native WebRTC mặc định dùng **Opus codec** — một codec được thiết kế đặc biệt cho speech realtime.

**Đặc điểm Opus:**
- **Sampling rate:** Hỗ trợ 5 tần suất: 8 kHz, 12 kHz, 16 kHz, 24 kHz, và **48 kHz** (phạm vi nghe của người)
- **Bitrate:** Tự động điều chỉnh (adaptive) từ 10 kbps đến 128+ kbps tùy chất lượng yêu cầu
- **Quality:** Toll-quality (MOS ~4.1-4.3), tương đương điện thoại truyền thống nhưng với tần số cao hơn
- **Bandwidth:** ~8,438 Hz (so với G.711 chỉ ~3,969 Hz)

**Ý nghĩa thực tế:** User sẽ nghe được:
- Những âm thâm tế (breathiness) từ hơi thở
- Âm sibilant (s, sh, ch) rõ ràng
- Biến đổi tông giọng tự nhiên
- Cảm giác gần gũi của giọng người nói

### Adaptive Bitrate — Thích ứng với mạng

Opus không có bitrate cố định. WebRTC liên tục theo dõi tình trạng mạng và tự điều chỉnh:

**TWCC (Transport-Wide Congestion Control):**
- Receiver ước tính bandwidth khả dụng dựa trên packet loss, jitter
- Gửi feedback về sender mỗi vài trăm milliseconds
- Sender tự động điều chỉnh bitrate của Opus (10 → 50 → 128 kbps)
- Kết quả: Không bị "trơ" tiếng khi mạng tạm khó khăn

**Practical implication:** Trên mạng 3G yếu, Opus sẽ tự giảm bitrate để giữ latency thấp. Trên mạng 4G/5G, nó sẽ tăng bitrate để nâng chất lượng. Không cần tuning thủ công.

### Echo Cancellation: AEC3

Browser có **tích hợp sẵn echo cancellation** (AEC3 của Google) qua MediaTrackSettings API.

**Vấn đề AEC giải quyết:**
- User nói vào microphone
- Loa của device phát TTS từ AI
- Microphone nghe lại tiếng loa phát (feedback)
- Kết quả: AI nghe thấy chính giọng nó, loop vô tận

**AEC3 hoạt động:**
- Học mô hình acoustic echo từ speaker → mic
- Trừ đi signal này khỏi microphone input
- Delay window: ~50ms (đủ cho local acoustic echo)
- Tự động áp dụng, user không cần config

**Quan trọng:** AEC3 là delay-agnostic, có nghĩa nó học và xóa echo ngay cả khi có latency biến đổi (jitter). Đây là lý do nó mạnh mẽ.

### Jitter Buffer: NetEQ

WebRTC tích hợp **NetEQ** — jitter buffer thông minh:

**Vấn đề jitter:**
- Network không truyền packets đều đặn
- Packet 1 đến lúc T, packet 2 đến lúc T+30ms, packet 3 đến lúc T+15ms
- Nghe tiếng lặp lại, gập gập, không mượt

**NetEQ tự động:**
- Monitor variation của packet arrival time
- Động điều chỉnh buffer depth (15-120ms)
- Xuyên suốt cân bằng: buffer lớn = ít gap nhưng latency cao, buffer nhỏ = latency thấp nhưng có gap
- Compression/expansion (time-scaling) khi packet missed — giãn hoặc nén audio để đủ thời gian

**Result:** Nghe natural dù network có jitter.

### Latency Characteristic

Với WebRTC browser:

**Típical end-to-end:** 150-250ms (một chiều)

Breakdown cho 200ms:
- Encoding Opus: 5-10ms
- Network round-trip: 50-100ms
- Jitter buffer absorption: 40-80ms
- Decoding: 5-10ms
- Audio playback: ~20ms

**Vì sao thấp hơn PSTN?**
- UDP transport (không cần TCP acknowledgment)
- Codec design tối ưu cho latency (Opus cấu trúc frame nhỏ)
- Peer-to-peer: không qua gateway

---

## Thế giới 2: Phone — PSTN, G.711, Limited Quality

### Audio Codec: G.711 ở 8 kHz

Khi user gọi vào số điện thoại của Voice AI system:

1. PSTN router của nhà cung cấp điện thoại xử lý
2. Kết nối đến Twilio gateway
3. Twilio stream audio qua WebSocket = **G.711 μ-law ở 8 kHz**

**Đặc điểm G.711:**
- **Sampling rate:** 8 kHz (cũ hơn, tiêu chuẩn từ thập niên 1970)
- **Bitrate:** Fixed 64 kbps (8-bit samples mỗi 1/8000 giây)
- **Network overhead:** ~80 kbps (16 kbps overhead cho framing)
- **Bandwidth:** ~3,969 Hz (hẹp — chỉ voice frequency)
- **Quality:** Toll-quality (MOS ~4.1-4.3), **nhưng** sẽ thiếu nuance

**Ý nghĩa thực tế:** User sẽ **không** nghe được:
- Subtle breathiness (hơi thở)
- Chi tiết sibilant (s, sh rõ ràng)
- Tone color tự nhiên (acoustic flavor)
- → Tiếng có cảm giác "máy" hơn, không tự nhiên

**Vì sao 8 kHz?** PSTN thiết kế từ thười Afrikaans (1950s), khi bandwidth là luxury. 8 kHz = 16 kHz Nyquist frequency = đủ cho speech intelligibility. Dù hôm nay bandwidth rẻ, PSTN vẫn "bị tóc" ở 8 kHz vì backward compatibility.

### Fixed Codec — Không Thích Ứng

Khác với Opus adaptive, G.711 là **fixed 64 kbps**, không bao giờ thay đổi.

**Ưu điểm:** Predictable — developer biết chính xác 80 kbps sẽ dùng.

**Nhược điểm:** Không cách nào giảm bitrate khi mạng tạm khó khăn. Nếu packet loss tăng, user nghe bị drop frame, không smooth.

### Echo Cancellation: PSTN Hybrid Echo

Khi user nghe AI TTS qua speaker phone, có 2 loại echo:

**1. Local Echo (từ browser WebRTC):**
- Loa phát → Microphone của device nghe lại
- AEC3 xử lý

**2. Far-End Echo (từ PSTN caller's phone):**
- **User gọi vào số điện thoại của AI system** (user là far-end)
- AI phát TTS ra loa
- Loa của user phát → Hybrid converter của user (4-wire PSTN → 2-wire = impedance mismatch)
- Reflection quay lại → WebSocket của Twilio → AI nhận lại echo

**Echo cancellation window:** PSTN hybrid echo cancellation handle ~64ms delay (tiêu chuẩn G.168).

Nhưng nếu AI TTS có latency cao hơn 64ms, PSTN echo canceller sẽ "miss" và echo sẽ trôi vào AI input, gây loop.

**Giải pháp:** Trong phone channel, AI phải detect echo path từ outbound audio (TTS của AI bị reflect về) và suppress nó trong inbound audio.

### PSTN Latency: <100ms Network, Nhưng Buffer Thêm

**PSTN network guarantee:** <100ms one-way (ITU-T G.114)

Nhưng:
- Twilio gateway buffering: +10-30ms
- Jitter buffer (giống WebRTC): +40-80ms
- Transcode overhead (Twilio phải convert G.711 ↔ PCM): +2-5ms

**Thực tế:** Phone channel có **total latency ~150-200ms** dù network chỉ ~100ms.

---

## So Sánh Trực Tiếp: Browser vs Phone

| Tiêu chí | Browser (WebRTC) | Phone (PSTN) |
|----------|------------------|--------------|
| **Codec** | Opus | G.711 μ-law |
| **Sample rate** | 48 kHz | 8 kHz |
| **Bitrate** | Adaptive 10-128+ kbps | Fixed 64 kbps + 16 overhead |
| **Audio bandwidth** | ~8,438 Hz (full speech) | ~3,969 Hz (speech only) |
| **Quality feel** | Natural, nuanced | Acceptable, but "tinny" |
| **Adaptive to network** | Yes (TWCC) | No (fixed) |
| **Echo cancellation** | AEC3 (local echo) | PSTN hybrid + AI detection |
| **Jitter buffer** | NetEQ (automatic) | Manual adaptive buffer |
| **Network latency** | 50-100ms | <100ms (PSTN) |
| **Total latency** | 150-250ms | 150-200ms |
| **Handling packet loss** | Smooth quality degrade | Quality cliff (packet drop) |
| **Architecture** | Peer-to-peer | Gateway + transcode |

---

## Preprocessing Pipeline: Nơi Hai Thế Giới Gặp Nhau

### Browser Path → AI Model

```
Microphone (48 kHz PCM)
    ↓
[Browser AEC3 + Noise Suppression]
    ↓
Opus Encode (adaptive bitrate)
    ↓
WebSocket → Server
    ↓
[Server: Opus Decode]
    ↓
Normalize to 16 kHz PCM (if needed for ASR)
    ↓
AI Model (STT + LLM + TTS)
    ↓
TTS Output (48 kHz Opus)
    ↓
WebSocket → Browser
    ↓
Decode + Audio Worklet → Speaker
```

**Key point:** Minimal preprocessing. Opus là 48 kHz, AI nhận 16 kHz, nhưng không cần đổi codec.

### Phone Path → AI Model

```
Twilio WebSocket (G.711 μ-law 8 kHz)
    ↓
[Server: G.711 Decode → PCM 8 kHz]
    ↓
Upsample 8 kHz → 16 kHz (Nyquist + speech recognition sweet spot)
    ↓
[VAD: Voice Activity Detection]
    ↓
[Barge-in: Cut TTS if user speaking]
    ↓
AI Model (STT + LLM + TTS generation)
    ↓
TTS Output (generated at ~24 kHz)
    ↓
Downsample 24 kHz → 8 kHz
    ↓
[Encode → G.711 μ-law]
    ↓
Twilio WebSocket → Phone
```

**Key point:** Heavy preprocessing. Receive 8 kHz → upsample → AI → downsample → 8 kHz. Mỗi lần resample là một cơ hội để tăng latency hoặc giảm chất lượng.

### Tại sao phải upsample 8 kHz → 16 kHz?

**Lý do 1: Speech Recognition (STT)**
- OpenAI Whisper, Google Cloud STT, all expect 16 kHz (hoặc cao hơn)
- 8 kHz là quá thấp — thiếu high-frequency consonant detail
- WER (Word Error Rate) tăng nếu feed 8 kHz vào STT model

**Lý do 2: LLM Voice Quality**
- AI text-to-speech model thường train ở 24 kHz (tự nhiên hơn)
- Nếu output TTS ở 8 kHz, tiếng sẽ méo mó, không tự nhiên
- Upsample 8 → 16 → pass to TTS model (output 24 kHz) → downsample → 8 kHz for PSTN

---

## Voice Activity Detection (VAD) & Barge-in

### VAD — Phát Hiện Người Nói

**Vấn đề:**
- User đang nói hay đang lắng nghe TTS?
- Nếu AI nói tiếp khi user nói, sẽ overlap nghe rất kinh khủng

**VAD (Voice Activity Detection):**
- Real-time signal processing: nghe inbound audio, detect speech vs silence
- Latency ~20-100ms (tùy algorithm)
- Accuracy: ~85-95% (tùy noise level)

**Trong Voice AI:**
1. AI generate TTS
2. Transmit TTS → User
3. Simultaneously monitor user inbound
4. Detect voice activity from user → cut TTS playback
5. Switch to listen mode

**Latency budget implication:**
- User speaks → VAD detects (~20-50ms) → AI receives → AI should cut TTS playback
- Ideal: <200ms (natural conversation feels like no delay)
- Reality: <600ms acceptable, >800ms feels broken

### Barge-in — User Nói Xen Ngang

Khi AI đang nói (TTS), user nói chen ngang:

**Điều gì xảy ra:**
1. **Phone channel:**
   - Inbound audio từ user: G.711 8 kHz
   - Outbound audio từ AI: G.711 8 kHz (encoder)
   - Cùng lúc: server nhận inbound từ user, gửi outbound từ AI
   
2. **VAD detects speech in inbound** → signal to stop TTS playback

3. **AI receives user's new input** → stop current response, re-prompt:
   ```
   "Ah, you interrupted. So you said [re-transcribe]. Let me answer that instead..."
   ```

**Technical challenge:**
- Timing: user speaks → latency → AI detects → latency → AI stops TTS
- Average: 200-300ms
- If VAD miss user speech, or latency too high, overlap happens (bad experience)

---

## Jitter Buffer & Adaptive Playout

### Jitter Problem

Network không cô lập packet theo thứ tự, và không đều thời gian:

```
Expected arrival: T, T+20ms, T+40ms, T+60ms
Actual arrival:   T, T+45ms, T+25ms, T+65ms (jitter)
```

Phát audio theo actual arrival → sẽ gập gập, không mượt.

### Jitter Buffer Solution

Collect packets vào buffer, play out theo regular schedule:

**Fixed buffer (old):**
```
20ms: collect packets 1-2, play packet 0
40ms: collect packets 3-4, play packet 1-2
60ms: collect packets 5-6, play packets 3-4
```
- Pro: Simple, deterministic latency
- Con: If heavy jitter, buffer overflow → packets lost, dropouts

**Adaptive buffer (modern):**
```
Monitor: (average_arrival_time - schedule) = buffer_delay
If delay increasing → expand buffer slightly
If delay stable → shrink buffer slightly
```
- Pro: Handle variable jitter, minimize buffer underflow
- Con: Algorithm complex, can oscillate if tuning bad

**Typical values:**
- Initial: 40-60ms
- Range: 15-120ms
- Goal: Minimize gaps without adding too much latency

---

## Preprocessing Challenges: Upsampling & Downsampling

### 8 kHz → 16 kHz Upsampling

**Concern:** Upsampling is just interpolation — không tạo ra thông tin mới.

**Thực tế:**
- Use high-quality interpolation filter (e.g., libsamplerate, SoX)
- Quality loss: **negligible** from resampling itself
- Quality loss is already done by G.711 codec (narrowband)
- Upsampling chỉ "spread" narrowband signal lên 16 kHz domain, không phục hồi lost high frequencies

**Example:**
- Original speech: 48 kHz full-spectrum
- PSTN encode: G.711 8 kHz (filter out >4 kHz, quantize to 8-bit)
- Twilio receive: 8 kHz
- Upsample to 16 kHz: 0-4 kHz content preserved, 4-8 kHz synthetic (all-zero)
- ASR still works because speech intelligibility preserved in 0-4 kHz

### 24 kHz → 8 kHz Downsampling

Opposite direction:

**Pipeline:** 
- TTS generate 24 kHz (natural-sounding)
- Downsample to 8 kHz (mandatory for PSTN)
- Encode to G.711

**Quality loss:**
- High frequencies (8-12 kHz sibilants) aliased down
- Tiếng sẽ "thiếu chi tiết" so với browser channel
- But intelligibility preserved (speech survives)

**Tip:** Must apply anti-aliasing low-pass filter <4 kHz before decimation, otherwise aliasing distortion.

---

## Tại Sao Kiến Trúc Phải Khác Nhau

### Browser Channel — Direct WebSocket Pattern

```
Browser ─→ Ephemeral Token ─→ Server (auth)
Browser ═══WSS═══════════════→ AI Model (direct)
         ←──────────────────
         (full Opus 48 kHz, adaptive, AEC, no transcode)
```

- Audio truyền thẳng 48 kHz
- Server không chạm audio → latency minimal
- Client side AEC → network không phải handle echo
- Adaptive bitrate → auto-adjust network quality

### Phone Channel — Double WebSocket + Transcode Pattern

```
Phone ──→ Twilio
         ════WSS#1════→ Server
                       [Transcode: G.711 ↔ PCM]
                       [Upsample: 8→16 kHz]
                       [VAD, Barge-in, Echo suppress]
                       [Downsample: 24→8 kHz]
                       ════WSS#2════→ AI Model
                       ←─────────────
                       (receive 16kHz, process, output TTS)
```

- Audio must flow through server for transcode
- Server handling: codec conversion, resampling, VAD, barge-in logic, echo suppression
- Latency addition từ server processing phải minimize

**Insight:** Hai channel không thể dùng cùng architecture vì audio format và processing pipeline quá khác nhau.

---

## Tóm Tắt — Những Gì Cần Nhớ

**Browser: Full-featured, high quality, complex networking**
- Opus 48 kHz adaptive codec
- Browser-native AEC3 + RNNoise suppression (optional)
- 150-250ms latency (peer-to-peer)
- No server processing overhead
- User experience: Natural, nuanced voice

**Phone: Legacy, fixed quality, predictable processing**
- G.711 8 kHz fixed codec
- PSTN echo path ~64ms window
- 150-200ms latency (via gateway)
- Server must handle transcode, VAD, barge-in, echo suppression
- User experience: Acceptable intelligibility, but "tinny" voice

**Preprocessing: Asymmetric paths**
- Browser: Minimal (Opus → PCM 16kHz → AI)
- Phone: Heavy (G.711 → upsample 8→16kHz → AI → downsample 24→8kHz → G.711)

**Architecture implication: Unified Server Pattern essential**
- Both channels must eventually reach one AI Model
- But input/output preprocessing is radically different
- Single process with shared ring buffers for WSS#1 + WSS#2 avoids latency trap
- This is why understanding WebSocket + channel differences sets up Bài 04 (Audio Bridge)

---

## Bài Tiếp Theo

Bài 04 — **Audio Bridge — Demo vs Production** sẽ giải quyết:
- Cách triển khai Audio Bridge thống nhất cho cả 2 channel
- Codec chaining: Opus → PCM → process → back to output codec
- Quality loss từ mỗi conversion
- Unified bridge pattern
- Cost/quality trade-offs

---

*Bài 3/15 · ← [WebSocket & Realtime](./02-websocket-realtime-communication.md) · [Roadmap](./README.md) → [Audio Bridge](./04-audio-bridge.md)*
