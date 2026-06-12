# BÀI KIỂM TRA NĂNG LỰC AI EDGE - SỐ 2
## Chủ đề: Tối ưu hóa tài nguyên Always-on và Tái tạo giọng đọc tự nhiên, đa ngôn ngữ (Code-switching).

## **Họ và tên:** Đỗ Tuấn Trực 

---

## Mục lục

1. [TỔNG QUAN HỆ THỐNG](#1-tổng-quan-hệ-thống)
2. [NHIỆM VỤ THIẾT KẾ (DESIGN TASKS)](#2-nhiệm-vụ-thiết-kế-design-tasks)
3. [KHỐI 1 — VAD & Ring Buffer](#3-khối-1--vad--ring-buffer)
4. [KHỐI 2 — Đa luồng Producer-Consumer](#4-khối-2--đa-luồng-producer-consumer)
5. [KHỐI 3 — Code-switching TTS](#5-khối-3--code-switching-tts)
6. [PSEUDO-CODE CHI TIẾT](#6-pseudo-code-chi-tiết)
7. [CÂU HỎI GIẢI TRÌNH (DOCUMENTATION)](#7-câu-hỏi-giải-trình-documentation)
8. [PHỤ LỤC](#8-phụ-lục)

---

## 1. TỔNG QUAN HỆ THỐNG

### 1.1 Bối cảnh

Hệ thống trợ lý ảo giám sát tích hợp trên bảng điều khiển xe điện thông minh phải đáp ứng ba yêu cầu cốt lõi:

| Yêu cầu | Mô tả | Ràng buộc |
|----------|--------|-----------|
| **Always-on Listening** | Microphone luôn mở, lắng nghe lệnh rảnh tay khi lái xe | CPU idle ≤ 40% |
| **Bilingual TTS** | Đọc cảnh báo kỹ thuật trộn lẫn Anh-Việt trong cùng câu | Model ≤ 100M params |
| **Real-time Response** | Phản hồi nhanh, không gây trễ cho hệ thống điều khiển xe | CPU active ≤ 70% |

### 1.2 Phần cứng đích

```
┌─────────────────────────────────────────────┐
│           Raspberry Pi 5                    │
│  ┌──────────────────────────────────────┐   │
│  │  Broadcom BCM2712                    │   │
│  │  4× ARM Cortex-A76 @ 2.4 GHz         │   │
│  │  RAM: 4GB / 8GB LPDDR4X              │   │
│  │  OS: Linux ARM64 (Debian Bookworm)   │   │
│  └──────────────────────────────────────┘   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ USB Mic  │  │ I2S DAC  │  │ CAN Bus  │   │
│  │ (Input)  │  │ (Output) │  │ (Vehicle)│   │
│  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────┘
```

### 1.3 Phân bổ CPU — Budget Allocation

```
      ┌──────────────────────────────────────────────┐
      │              TỔNG CPU = 100%                 │
      ├────────────────────┬─────────────────────────┤
      │   CHẾ ĐỘ CHỜ       │   CHẾ ĐỘ ACTIVE         │
      ├────────────────────┼─────────────────────────┤
      │ Audio+VAD: ≤15%    │ Audio+VAD:    ~15%      │
      │ Overhead:  ≤5%     │ ASR Infer:    ~35%      │
      │ Dashboard: ~20%    │ TTS Infer:    ~15%      │
      │ Motor Ctrl: ~30%   │ Dashboard:    ~10%      │
      │ Reserved:  ~30%    │ Motor Ctrl:   ~20%      │
      │                    │ Overhead:     ~5%       │
      │ TOTAL VOICE: ≤40%  │ TOTAL VOICE:  ≤70%      │
      └────────────────────┴─────────────────────────┘
```

---

## 2. NHIỆM VỤ THIẾT KẾ (DESIGN TASKS)

### 2.1 Sơ đồ luồng dữ liệu (Data Flow)

```
 ┌─────────┐    ┌────────────┐    ┌──────────────┐    ┌─────────┐
 │   MIC   │───▶│ Ring Buffer│───▶│  Silero VAD  │───▶│  Queue │
 │ (16kHz) │    │ (3s fixed) │    │ (ONNX, ~2MB) │    │(maxsize)│
 └─────────┘    └────────────┘    └──────────────┘    └────┬────┘
                                                           │
         ┌─────────────────────────────────────────────────┘
         │                              
         ▼                              
 ┌───────────────┐    ┌──────────────────┐    ┌─────────────────┐
 │ SenseVoice    │───▶│ Text Normalizer  │───▶│  Valtec-TTS     │
 │ ASR (INT8)    │    │ + Phonemizer     │    │  + Prosody Ctrl  │
 │ ~234M → ~60M  │    │ (Regex/Rules)    │    │  (~74.8M params) │
 └───────────────┘    └──────────────────┘    └───────┬─────────┘
                                                       │
                                                       ▼
                                               ┌──────────────┐
                                               │   Speaker    │
                                               │  (I2S DAC)   │
                                               └──────────────┘
```

### 2.2 Mô hình Luồng (Threading Model)

```
 ┌──────────────────────────────────────────────────────────────┐
 │                    PROCESS: ev_voiceguard                    │
 │                                                              │
 │  ┌──────────────────┐         ┌──────────────────────────┐   │
 │  │  THREAD 1         │        │  THREAD 2                 │  │
 │  │  (Producer)       │        │  (Consumer)               │  │
 │  │  CPU: Core 0      │        │  CPU: Core 1-2            │  │
 │  │                   │        │                           │  │
 │  │  Audio Capture    │        │  ┌─── ASR Inference ───┐  │  │
 │  │       ↓           │        │  │ SenseVoiceSmall     │  │  │
 │  │  Ring Buffer      │  Queue │  │ (wake on data)      │  │  │
 │  │       ↓           │───────▶│  └─────────────────────┘  │  │
 │  │  VAD Filter       │        │           ↓               │  │
 │  │       ↓           │        │  ┌─── TTS Pipeline ───┐   │  │
 │  │  Push if speech   │        │  │ Normalize → Phone   │  │  │
 │  │                   │        │  │ → Synthesize → Play │  │  │
 │  └──────────────────┘         │  └─────────────────────┘  │  │
 │                               └──────────────────────────┘   │
 │                                                              │
 │  ┌──────────────────┐                                        │
 │  │  THREAD 3         │                                       │
 │  │  (Monitor)        │                                       │
 │  │  CPU Governor     │                                       │
 │  │  Health Check     │                                       │
 │  └──────────────────┘                                        │
 └──────────────────────────────────────────────────────────────┘
```

---

## 3. KHỐI 1 — VAD & Ring Buffer

### 3.1 Ring Buffer: Thiết kế Chi tiết

#### Tại sao Ring Buffer?

| Approach | Bộ nhớ sau 1h | Bộ nhớ sau 24h | An toàn? |
|----------|---------------|----------------|----------|
| `list.append()` | ~225 MB | ~5.4 GB | ❌ OOM crash |
| Ring Buffer (3s) | ~96 KB | ~96 KB | ✅ Cố định |

#### Cơ chế hoạt động

```
Ring Buffer (capacity = 3 giây = 48,000 samples @ 16kHz)

Trạng thái ban đầu:
┌───┬───┬───┬───┬───┬───┬───┬───┐
│   │   │   │   │   │   │   │   │  write_pos = 0
└───┴───┴───┴───┴───┴───┴───┴───┘

Sau khi ghi 5 chunk:
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ A │ B │ C │ D │ E │   │   │   │  write_pos = 5
└───┴───┴───┴───┴───┴───┴───┴───┘

Buffer đầy, ghi đè vòng:
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ I │ J │ C │ D │ E │ F │ G │ H │  write_pos = 2 (vòng lại)
└───┴───┴───┴───┴───┴───┴───┴───┘
  ↑                               
  write_pos (chunk cũ nhất bị ghi đè)
```

#### Thông số kỹ thuật

```python
# Tính toán kích thước buffer
SAMPLE_RATE    = 16000      # 16 kHz
SAMPLE_WIDTH   = 2          # 16-bit PCM = 2 bytes/sample
BUFFER_SECONDS = 3          # Lưu 3 giây gần nhất
CHUNK_DURATION = 0.03       # 30ms per chunk (khuyến nghị Silero VAD)
CHUNK_SIZE     = int(SAMPLE_RATE * CHUNK_DURATION)  # 480 samples
BUFFER_CHUNKS  = int(BUFFER_SECONDS / CHUNK_DURATION)  # 100 chunks

# Tổng bộ nhớ: 480 samples × 2 bytes × 100 chunks = 96,000 bytes ≈ 94 KB
```

### 3.2 Silero VAD: Tích hợp

#### Lý do chọn Silero VAD thay vì WebRTC VAD

| Tiêu chí | Silero VAD | WebRTC VAD |
|----------|-----------|------------|
| Kích thước model | ~2 MB (ONNX) | ~100 KB (C lib) |
| Độ chính xác (noisy) | **Cao** — trained on diverse noise | Trung bình |
| Latency per chunk | ~1-3 ms | <1 ms |
| Adaptive threshold | ✅ Có | ❌ Cố định (aggressiveness 0-3) |
| CPU usage (ARM64) | ~2-5% | ~1-2% |
| Phù hợp môi trường xe | **✅ Tốt hơn** (gió, động cơ, còi) | Kém hơn |

**Kết luận:** Silero VAD được ưu tiên vì độ chính xác vượt trội trong môi trường ồn ào của xe. WebRTC VAD dùng làm fallback khi cần giảm CPU tối đa.

#### Cấu hình VAD cho môi trường xe

```python
VAD_CONFIG = {
    "threshold": 0.5,          # Ngưỡng xác suất giọng nói (0.0 - 1.0)
    "min_speech_duration": 0.25, # Tối thiểu 250ms để xác nhận speech
    "min_silence_duration": 0.5, # 500ms im lặng → kết thúc utterance
    "speech_pad_duration": 0.3,  # Padding 300ms trước/sau speech
    "window_size_samples": 512,  # Kích thước cửa sổ phân tích
    "max_speech_duration": 30.0, # Timeout tối đa 30 giây
}
```

### 3.3 Quy trình xử lý Audio → VAD

```
1. PyAudio mở stream (16kHz, mono, 16-bit PCM)
2. Đọc chunk 30ms (480 samples) từ mic
3. Ghi chunk vào Ring Buffer (ghi đè chunk cũ nhất nếu đầy)
4. Đẩy chunk qua Silero VAD
5. VAD trả về probability (0.0 - 1.0)
   ├── prob < threshold → Bỏ qua (tiếng ồn)
   └── prob ≥ threshold → Đánh dấu "speech detected"
6. Nếu speech phát hiện:
   ├── Bắt đầu tích lũy chunks vào speech_buffer
   ├── Thêm padding chunks trước đó (từ Ring Buffer)
   └── Tiếp tục cho đến khi min_silence_duration đạt
7. Khi utterance hoàn tất → Đẩy speech_buffer vào Queue
```

---

## 4. KHỐI 2 — Đa luồng Producer-Consumer

### 4.1 Thiết kế Thread-safe Queue

#### Yêu cầu

- **Thread-safe:** Nhiều thread truy cập đồng thời không gây race condition.
- **Bounded:** Giới hạn kích thước (`maxsize`) để ngăn tràn RAM.
- **Blocking Consumer:** Thread 2 ngủ khi queue rỗng, tự thức dậy khi có dữ liệu.
- **Non-blocking Producer:** Thread 1 không bao giờ bị block cứng.

#### Cơ chế Backpressure

```
Queue Status:      [■ ■ ■ ■ ■ ■ ■ ■ ■ ■]  ← ĐẦY (maxsize=10)
                                              
Producer muốn đẩy thêm?                      
  ↓                                           
Chiến lược DROP-OLDEST:                       
  1. Loại bỏ item cũ nhất trong queue        
  2. Đẩy item mới vào                        
  3. Log cảnh báo "Frame dropped"            
  4. Tăng counter drop_count                  

Tác dụng phụ:
  - Mất một phần đầu câu nói (chấp nhận được)
  - Tránh hệ thống bị treo hoàn toàn (critical)
  - User có thể cần lặp lại câu lệnh (minor UX impact)
```

### 4.2 Cơ chế Wake-up (Condition Variable)

```python
# Thread 2 KHÔNG dùng busy-wait (while True: check queue)
# Thay vào đó, dùng queue.Queue.get(timeout=...)

# Lý do: busy-wait sẽ chiếm CPU liên tục ngay cả khi idle
# queue.get() sử dụng condition variable nội bộ → Thread ngủ thật sự

# So sánh:
# ❌ Busy-wait:  while queue.empty(): pass    → CPU 100% khi idle
# ✅ Blocking:   item = queue.get(timeout=1)  → CPU ~0% khi idle
```

### 4.3 CPU Affinity Pinning

```python
import os

def pin_thread_to_cores(cores: list[int]):
    """Ghim thread hiện tại vào các CPU cores chỉ định."""
    pid = os.getpid()
    os.sched_setaffinity(pid, set(cores))

# Thread 1 (Audio + VAD): Core 0 — chạy liên tục, nhẹ
# Thread 2 (ASR + TTS):   Core 1, 2 — burst khi có speech
# Core 3: Reserved cho Motor Control + Dashboard
```

### 4.4 Sơ đồ trạng thái (State Machine)

```
                    ┌──────────┐
                    │  IDLE    │ ← Thread 2 ngủ
                    │ (0% CPU) │
                    └────┬─────┘
                         │ Queue.get() nhận data
                         ▼
                    ┌──────────┐
                    │ ASR      │ ← Thread 2 thức
                    │ RUNNING  │   (~35% CPU)
                    │          │
                    └────┬─────┘
                         │ ASR trả về text
                         ▼
                    ┌──────────┐
                    │ TTS      │ ← Chuyển sang TTS
                    │ SPEAKING │   (~15% CPU)
                    │          │
                    └────┬─────┘
                         │ Phát xong audio
                         ▼
                    ┌──────────┐
                    │  IDLE    │ ← Quay lại ngủ
                    └──────────┘
```

---

## 5. KHỐI 3 — Code-switching TTS

### 5.1 Bài toán Code-switching

**Input mẫu:**
```
"Hệ thống đang kiểm tra BMS, phát hiện lỗi Overcurrent trên đường nguồn 24V"
"Mã lỗi CAN bus communication timeout"
"Battery SOC còn 15%, khuyến nghị sạc DC fast charging"
```

**Thách thức:** Mô hình TTS nhỏ (~74.8M params) không có đủ từ vựng tiếng Anh để phát âm chính xác các thuật ngữ kỹ thuật.

### 5.2 Giải pháp: Dual-layer Phonemizer

Áp dụng chiến lược **2 tầng xử lý** trước khi đưa vào model TTS:

```
Input Text
    │
    ▼
┌─────────────────────────────────────────┐
│  TẦNG 1: Text Normalization (Regex)     │
│                                         │
│  1. Viết tắt → Phiên âm Việt            │
│     "BMS"    → "bi em ét"               │
│     "SOC"    → "ét ô xi"                │
│     "CAN"    → "can" (giữ nguyên)       │
│     "DC"     → "đi xi"                  │
│     "24V"    → "hai mươi bốn vôn"       │
│                                         │
│  2. Thuật ngữ → Phiên âm IPA-Việt       │
│     "Overcurrent" → "ô-vơ-cơ-rần"       │
│     "timeout"     → "thai-ao"           │
│     "fast charging"→ "phát chác-jinh"   │
│                                         │
│  3. Số / Đơn vị → Dạng chữ              │
│     "15%"    → "mười lăm phần trăm"     │
│     "100A"   → "một trăm am-pe"         │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│  TẦNG 2: Language-tagged Phoneme Router │
│                                         │
│  Nếu Tầng 1 không cover → phát hiện     │
│  ngôn ngữ từng segment:                 │
│                                         │
│  "[VI] Hệ thống đang kiểm tra           │
│   [EN→VI_PHONEME] bi em ét,             │
│   [VI] phát hiện lỗi                    │
│   [EN→VI_PHONEME] ô-vơ-cơ-rần           │
│   [VI] trên đường nguồn                 │
│   [VI] hai mươi bốn vôn"                │
│                                         │
│  → Gộp thành chuỗi thuần Việt           │
│  → Đẩy vào TTS model                    │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│  Valtec-TTS / VieNeu-TTS                │
│  Input: Chuỗi thuần Vietnamese phonemes │
│  Output: Waveform audio                 │
└─────────────────────────────────────────┘
```

### 5.3 Cấu trúc Lexicon File

```
# lexicon/en_abbreviations.dict
# Format: PATTERN → REPLACEMENT
BMS         bi em ét
SOC         ét ô xi
CAN         can
DC          đi xi
AC          ây xi
ECU         i xi iu
OBD         ô bi đi
GPS         ji pi ét
USB         iu ét bi
LED         lét
RPM         ác pi em

# lexicon/vi_technical.dict
# Format: ENGLISH_TERM → VIETNAMESE_PHONETIC
overcurrent     ô-vơ-cơ-rần
overvoltage     ô-vơ-vôn-tịch
undervoltage    ân-đờ-vôn-tịch
timeout         thai-ao
communication   cơm-miu-ni-kây-sần
fast charging   phát chác-jinh
battery         bát-tơ-ri
controller      cơn-trô-lơ
inverter        in-vơ-tơ
sensor          xen-xơ
module          mô-đun
firmware        phơm-que
```

### 5.4 So sánh Phương án

| Tiêu chí | Regex/Rules (Text-norm) | Can thiệp Lexicon/Tokenizer |
|----------|------------------------|----------------------------|
| **Tốc độ CPU** | ⚡ Cực nhanh (<1ms) | 🐢 Chậm hơn (cần rebuild model) |
| **Linh hoạt** | ✅ Thêm từ mới = thêm 1 dòng dict | ❌ Cần retrain/finetune |
| **Kích thước model** | ✅ Không tăng | ❌ Tăng (thêm embeddings) |
| **Chất lượng phát âm** | ⚠️ Phụ thuộc phiên âm thủ công | ✅ Tự nhiên hơn |
| **Bảo trì** | ✅ Dễ (file text) | ❌ Khó (cần ML pipeline) |
| **Phù hợp embedded** | ✅ Rất phù hợp | ❌ Không phù hợp |

**→ Kết luận: Chọn Regex/Rules (Text-normalization)** vì phù hợp nhất với ràng buộc phần cứng nhúng. Chấp nhận trade-off chất lượng phát âm kém tự nhiên hơn một chút nhưng đổi lại tốc độ xử lý cực nhanh và dễ bảo trì.

### 5.5 Prosody Control (Kiểm soát Ngữ điệu)

#### Cơ chế điều khiển Pitch/Speed theo ngữ cảnh

```python
# Phân loại mức độ cảnh báo → Ánh xạ prosody parameters
PROSODY_PROFILES = {
    "normal": {
        "speed": 1.0,      # Tốc độ bình thường
        "pitch": 1.0,      # Cao độ bình thường
        "energy": 1.0,     # Năng lượng bình thường
    },
    "warning": {
        "speed": 1.15,     # Nhanh hơn 15%
        "pitch": 1.1,      # Cao hơn 10%
        "energy": 1.2,     # Mạnh hơn 20%
    },
    "critical": {
        "speed": 1.3,      # Nhanh hơn 30%
        "pitch": 1.25,     # Cao hơn 25%
        "energy": 1.4,     # Mạnh hơn 40%
    },
}
```

#### Cách triển khai không tăng inference time

```
Phương án 1: Post-processing (Khuyến nghị)
──────────────────────────────────────────
TTS Model → Waveform (tốc độ/pitch gốc)
         → librosa.effects.time_stretch(rate=speed)
         → librosa.effects.pitch_shift(sr, y, n_steps=pitch)
         → Output

Ưu điểm: Không chạm vào model, xử lý ở tầng DSP
Nhược điểm: Chất lượng giảm nhẹ khi stretch quá mức

Phương án 2: Conditioning Input (Nếu model hỗ trợ)
──────────────────────────────────────────
Một số model TTS (như Valtec-TTS) hỗ trợ duration/pitch
conditioning vectors. Điều chỉnh trực tiếp tại inference:
  - duration_scale = 1.0 / speed  (nói nhanh = duration ngắn)
  - pitch_scale = pitch
→ Không tăng compute, chỉ thay đổi input tensor
```

---

## 6. PSEUDO-CODE CHI TIẾT

### 6.1 Module Ring Buffer

```python
import numpy as np
from typing import Optional

class RingBuffer:
    """
    Bộ đệm vòng kích thước cố định cho dữ liệu âm thanh.
    Tự động ghi đè dữ liệu cũ nhất khi đầy.
    Bộ nhớ cố định ~94KB bất kể thời gian chạy.
    """
    
    def __init__(self, max_chunks: int = 100, chunk_size: int = 480):
        self._buffer = np.zeros((max_chunks, chunk_size), dtype=np.int16)
        self._max_chunks = max_chunks
        self._chunk_size = chunk_size
        self._write_pos = 0       # Vị trí ghi tiếp theo
        self._count = 0           # Số chunk đã ghi (tối đa = max_chunks)
    
    def write(self, chunk: np.ndarray) -> None:
        """Ghi 1 chunk vào buffer. Ghi đè chunk cũ nhất nếu đầy."""
        assert len(chunk) == self._chunk_size, \
            f"Chunk size mismatch: expected {self._chunk_size}, got {len(chunk)}"
        
        self._buffer[self._write_pos] = chunk
        self._write_pos = (self._write_pos + 1) % self._max_chunks
        self._count = min(self._count + 1, self._max_chunks)
    
    def read_last_n(self, n: int) -> Optional[np.ndarray]:
        """Đọc n chunk gần nhất theo thứ tự thời gian."""
        if n > self._count:
            n = self._count
        if n == 0:
            return None
        
        indices = []
        for i in range(n):
            idx = (self._write_pos - n + i) % self._max_chunks
            indices.append(idx)
        
        return self._buffer[indices].flatten()
    
    def clear(self) -> None:
        """Reset buffer."""
        self._write_pos = 0
        self._count = 0
        self._buffer.fill(0)
    
    @property
    def memory_usage_bytes(self) -> int:
        """Trả về dung lượng bộ nhớ cố định của buffer."""
        return self._buffer.nbytes  # Luôn = max_chunks × chunk_size × 2
```

### 6.2 Pipeline Producer-Consumer

```python
import threading
import queue
import time
import logging
import numpy as np
from enum import Enum, auto

logger = logging.getLogger("ev_voiceguard")


class PipelineState(Enum):
    IDLE = auto()
    LISTENING = auto()
    PROCESSING = auto()
    SPEAKING = auto()
    ERROR = auto()


class AlwaysOnPipeline:
    """
    Pipeline chính: Producer (Audio+VAD) → Queue → Consumer (ASR+TTS).
    Thiết kế cho hoạt động 24/7 trên phần cứng nhúng.
    """
    
    def __init__(self, config: dict):
        # ── Thread-safe Queue với giới hạn kích thước ──
        self.audio_queue = queue.Queue(maxsize=config.get("queue_maxsize", 10))
        
        # ── Ring Buffer ──
        self.ring_buffer = RingBuffer(
            max_chunks=config.get("buffer_chunks", 100),
            chunk_size=config.get("chunk_size", 480),
        )
        
        # ── State management ──
        self._state = PipelineState.IDLE
        self._state_lock = threading.Lock()
        self._running = threading.Event()
        self._running.set()  # Bắt đầu ở trạng thái chạy
        
        # ── Thống kê ──
        self._stats = {
            "frames_captured": 0,
            "frames_dropped": 0,
            "vad_triggers": 0,
            "asr_inferences": 0,
            "tts_syntheses": 0,
        }
        self._stats_lock = threading.Lock()
        
        # ── Config ──
        self._config = config
        self._sample_rate = config.get("sample_rate", 16000)
        self._vad_threshold = config.get("vad_threshold", 0.5)
        self._max_speech_duration = config.get("max_speech_duration", 30.0)
        self._min_silence_duration = config.get("min_silence_duration", 0.5)
        self._speech_pad_chunks = config.get("speech_pad_chunks", 10)
        
        # ── Models (lazy-loaded) ──
        self._vad_model = None
        self._asr_model = None
        self._tts_engine = None
    
    # ──────────────────────────────────────────────
    #  INITIALIZATION
    # ──────────────────────────────────────────────
    
    def _load_models(self):
        """Tải models vào RAM. Gọi 1 lần khi khởi động."""
        logger.info("Loading VAD model (Silero ONNX)...")
        # self._vad_model = SileroVAD("models/silero_vad.onnx")
        
        logger.info("Loading ASR model (SenseVoiceSmall INT8)...")
        # self._asr_model = SenseVoiceASR("models/sensevoice_small_int8.onnx")
        
        logger.info("Loading TTS engine (Valtec-TTS)...")
        # self._tts_engine = ValtecTTS("models/valtec_tts.onnx")
        
        logger.info("All models loaded successfully.")
    
    # ──────────────────────────────────────────────
    #  THREAD 1: PRODUCER (Audio Capture + VAD)
    # ──────────────────────────────────────────────
    
    def producer_audio_vad_thread(self):
        """
        Thread 1: Thu âm liên tục → Ring Buffer → VAD → Queue.
        Pinned to Core 0. CPU target: ≤15%.
        """
        # Ghim vào Core 0
        self._pin_to_cores([0])
        logger.info("Producer thread started on Core 0")
        
        # Khởi tạo PyAudio stream
        # stream = pyaudio.PyAudio().open(
        #     format=pyaudio.paInt16,
        #     channels=1,
        #     rate=self._sample_rate,
        #     input=True,
        #     frames_per_buffer=self._config["chunk_size"],
        # )
        
        speech_buffer = []       # Tạm chứa chunks của utterance hiện tại
        is_speaking = False      # Trạng thái: đang trong speech segment?
        silence_counter = 0      # Đếm chunk im lặng liên tiếp
        speech_counter = 0       # Đếm chunk speech liên tiếp
        
        silence_threshold = int(
            self._min_silence_duration / (self._config["chunk_size"] / self._sample_rate)
        )  # Số chunks im lặng để kết thúc utterance
        
        max_speech_chunks = int(
            self._max_speech_duration / (self._config["chunk_size"] / self._sample_rate)
        )  # Timeout: tối đa 30 giây speech liên tục
        
        while self._running.is_set():
            try:
                # ── Bước 1: Đọc chunk từ microphone ──
                # raw_data = stream.read(self._config["chunk_size"], 
                #                        exception_on_overflow=False)
                # chunk = np.frombuffer(raw_data, dtype=np.int16)
                chunk = np.zeros(self._config["chunk_size"], dtype=np.int16)  # Placeholder
                
                # ── Bước 2: Ghi vào Ring Buffer ──
                self.ring_buffer.write(chunk)
                
                with self._stats_lock:
                    self._stats["frames_captured"] += 1
                
                # ── Bước 3: Chạy VAD ──
                # vad_prob = self._vad_model.predict(chunk)
                vad_prob = 0.0  # Placeholder
                is_speech = vad_prob >= self._vad_threshold
                
                # ── Bước 4: State machine cho speech detection ──
                if is_speech:
                    silence_counter = 0
                    speech_counter += 1
                    
                    if not is_speaking:
                        # Bắt đầu speech mới
                        is_speaking = True
                        
                        with self._stats_lock:
                            self._stats["vad_triggers"] += 1
                        
                        # Lấy padding chunks từ Ring Buffer (âm thanh trước speech)
                        padding = self.ring_buffer.read_last_n(self._speech_pad_chunks)
                        if padding is not None:
                            speech_buffer.append(padding)
                        
                        logger.debug(f"Speech detected (prob={vad_prob:.2f})")
                    
                    speech_buffer.append(chunk.copy())
                    
                    # ── Timeout: speech quá dài ──
                    if speech_counter >= max_speech_chunks:
                        logger.warning(
                            f"Speech timeout ({self._max_speech_duration}s). "
                            "Forcing utterance end."
                        )
                        self._flush_speech_buffer(speech_buffer)
                        speech_buffer = []
                        is_speaking = False
                        speech_counter = 0
                
                else:  # Silence
                    if is_speaking:
                        silence_counter += 1
                        speech_buffer.append(chunk.copy())
                        
                        # Đủ im lặng → Kết thúc utterance
                        if silence_counter >= silence_threshold:
                            logger.debug(
                                f"Utterance complete. "
                                f"Duration: {len(speech_buffer)} chunks"
                            )
                            self._flush_speech_buffer(speech_buffer)
                            speech_buffer = []
                            is_speaking = False
                            speech_counter = 0
                            silence_counter = 0
            
            except Exception as e:
                logger.error(f"Producer error: {e}")
                time.sleep(0.1)  # Tránh tight loop khi có lỗi
        
        logger.info("Producer thread stopped.")
    
    def _flush_speech_buffer(self, speech_buffer: list):
        """
        Đẩy speech buffer vào Queue một cách an toàn.
        Áp dụng backpressure: drop oldest nếu queue đầy.
        """
        if not speech_buffer:
            return
        
        audio_data = np.concatenate(speech_buffer)
        
        try:
            # Non-blocking put: nếu queue đầy → xử lý backpressure
            self.audio_queue.put_nowait(audio_data)
        except queue.Full:
            # ── BACKPRESSURE: Drop oldest item ──
            try:
                dropped = self.audio_queue.get_nowait()
                with self._stats_lock:
                    self._stats["frames_dropped"] += 1
                logger.warning(
                    f"Queue full! Dropped oldest frame "
                    f"({len(dropped)} samples). "
                    f"Total drops: {self._stats['frames_dropped']}"
                )
            except queue.Empty:
                pass
            
            # Thử lại sau khi drop
            try:
                self.audio_queue.put_nowait(audio_data)
            except queue.Full:
                logger.error("Queue still full after drop. Data lost.")
                with self._stats_lock:
                    self._stats["frames_dropped"] += 1
    
    # ──────────────────────────────────────────────
    #  THREAD 2: CONSUMER (ASR + TTS)
    # ──────────────────────────────────────────────
    
    def consumer_asr_thread(self):
        """
        Thread 2: Lấy audio từ Queue → ASR → Text Processing → TTS.
        Pinned to Core 1-2. CPU target: ≤35% (ASR) + ≤15% (TTS).
        Thread này NGỦ khi queue rỗng (không busy-wait).
        """
        # Ghim vào Core 1 và Core 2
        self._pin_to_cores([1, 2])
        logger.info("Consumer thread started on Core 1-2")
        
        while self._running.is_set():
            try:
                # ── Blocking get với timeout ──
                # Thread NGỦ ở đây cho đến khi có dữ liệu hoặc timeout
                # Timeout 1 giây: cho phép kiểm tra self._running định kỳ
                audio_data = self.audio_queue.get(timeout=1.0)
                
            except queue.Empty:
                # Timeout → quay lại kiểm tra self._running
                continue
            
            try:
                self._set_state(PipelineState.PROCESSING)
                
                # ── Bước 1: ASR Inference ──
                logger.info(f"Running ASR on {len(audio_data)} samples...")
                # transcript = self._asr_model.transcribe(audio_data)
                transcript = ""  # Placeholder
                
                with self._stats_lock:
                    self._stats["asr_inferences"] += 1
                
                if not transcript.strip():
                    logger.debug("ASR returned empty transcript. Skipping.")
                    self._set_state(PipelineState.IDLE)
                    continue
                
                logger.info(f"ASR result: '{transcript}'")
                
                # ── Bước 2: Text Normalization (Code-switching) ──
                normalized_text = self._normalize_bilingual_text(transcript)
                logger.debug(f"Normalized: '{normalized_text}'")
                
                # ── Bước 3: Xác định Prosody Profile ──
                prosody = self._detect_alert_level(normalized_text)
                
                # ── Bước 4: TTS Synthesis ──
                self._set_state(PipelineState.SPEAKING)
                logger.info(f"TTS synthesizing with prosody={prosody['profile']}...")
                # audio_out = self._tts_engine.synthesize(
                #     text=normalized_text,
                #     speed=prosody["speed"],
                #     pitch=prosody["pitch"],
                # )
                # self._play_audio(audio_out)
                
                with self._stats_lock:
                    self._stats["tts_syntheses"] += 1
                
            except Exception as e:
                logger.error(f"Consumer processing error: {e}")
            
            finally:
                self._set_state(PipelineState.IDLE)
                self.audio_queue.task_done()
        
        logger.info("Consumer thread stopped.")
    
    # ──────────────────────────────────────────────
    #  TEXT NORMALIZATION (Code-switching handler)
    # ──────────────────────────────────────────────
    
    def _normalize_bilingual_text(self, text: str) -> str:
        """
        Chuẩn hóa text đa ngôn ngữ Anh-Việt:
        1. Thay thế viết tắt bằng phiên âm Việt
        2. Chuyển thuật ngữ Anh sang âm Việt
        3. Chuẩn hóa số và đơn vị
        """
        import re
        
        # Bước 1: Load lexicon (cached)
        abbreviations = self._load_lexicon("lexicon/en_abbreviations.dict")
        technical_terms = self._load_lexicon("lexicon/vi_technical.dict")
        
        # Bước 2: Thay viết tắt (case-insensitive, whole word)
        for abbr, phonetic in abbreviations.items():
            pattern = r'\b' + re.escape(abbr) + r'\b'
            text = re.sub(pattern, phonetic, text, flags=re.IGNORECASE)
        
        # Bước 3: Thay thuật ngữ kỹ thuật
        for term, phonetic in technical_terms.items():
            pattern = r'\b' + re.escape(term) + r'\b'
            text = re.sub(pattern, phonetic, text, flags=re.IGNORECASE)
        
        # Bước 4: Chuẩn hóa số + đơn vị
        # "24V" → "hai mươi bốn vôn"
        text = re.sub(
            r'(\d+)\s*V\b',
            lambda m: self._number_to_vietnamese(int(m.group(1))) + " vôn",
            text
        )
        text = re.sub(
            r'(\d+)\s*A\b',
            lambda m: self._number_to_vietnamese(int(m.group(1))) + " am-pe",
            text
        )
        text = re.sub(
            r'(\d+)\s*%',
            lambda m: self._number_to_vietnamese(int(m.group(1))) + " phần trăm",
            text
        )
        text = re.sub(
            r'(\d+)\s*°C\b',
            lambda m: self._number_to_vietnamese(int(m.group(1))) + " độ xê",
            text
        )
        
        return text
    
    # ──────────────────────────────────────────────
    #  PROSODY DETECTION
    # ──────────────────────────────────────────────
    
    def _detect_alert_level(self, text: str) -> dict:
        """
        Phát hiện mức độ cảnh báo từ nội dung text.
        Trả về prosody profile tương ứng.
        """
        critical_keywords = ["lỗi nghiêm trọng", "khẩn cấp", "nguy hiểm", 
                           "cháy", "rò rỉ", "mất phanh", "critical"]
        warning_keywords = ["cảnh báo", "lỗi", "phát hiện", "bất thường",
                          "warning", "quá nhiệt", "quá dòng"]
        
        text_lower = text.lower()
        
        if any(kw in text_lower for kw in critical_keywords):
            return {
                "profile": "critical",
                "speed": 1.3,
                "pitch": 1.25,
                "energy": 1.4,
            }
        elif any(kw in text_lower for kw in warning_keywords):
            return {
                "profile": "warning",
                "speed": 1.15,
                "pitch": 1.1,
                "energy": 1.2,
            }
        else:
            return {
                "profile": "normal",
                "speed": 1.0,
                "pitch": 1.0,
                "energy": 1.0,
            }
    
    # ──────────────────────────────────────────────
    #  UTILITY METHODS
    # ──────────────────────────────────────────────
    
    def _set_state(self, state: PipelineState):
        with self._state_lock:
            self._state = state
    
    def _pin_to_cores(self, cores: list):
        """Ghim thread hiện tại vào CPU cores chỉ định (Linux only)."""
        try:
            import os
            os.sched_setaffinity(0, set(cores))
        except (AttributeError, OSError):
            logger.warning("CPU affinity pinning not supported on this platform.")
    
    def _load_lexicon(self, path: str) -> dict:
        """Load lexicon file. Cached sau lần đầu."""
        if not hasattr(self, '_lexicon_cache'):
            self._lexicon_cache = {}
        if path in self._lexicon_cache:
            return self._lexicon_cache[path]
        
        lexicon = {}
        try:
            with open(path, 'r', encoding='utf-8') as f:
                for line in f:
                    line = line.strip()
                    if not line or line.startswith('#'):
                        continue
                    parts = line.split(maxsplit=1)
                    if len(parts) == 2:
                        lexicon[parts[0].lower()] = parts[1]
        except FileNotFoundError:
            logger.warning(f"Lexicon file not found: {path}")
        
        self._lexicon_cache[path] = lexicon
        return lexicon
    
    @staticmethod
    def _number_to_vietnamese(n: int) -> str:
        """Chuyển số sang chữ tiếng Việt (đơn giản, hỗ trợ 0-9999)."""
        if n == 0:
            return "không"
        
        ones = ["", "một", "hai", "ba", "bốn", "năm", 
                "sáu", "bảy", "tám", "chín"]
        
        if n < 10:
            return ones[n]
        elif n < 100:
            tens = n // 10
            unit = n % 10
            result = ones[tens] + " mươi"
            if unit == 1 and tens > 1:
                result += " mốt"
            elif unit == 5 and tens > 0:
                result += " lăm"
            elif unit > 0:
                result += " " + ones[unit]
            return result
        elif n < 1000:
            hundreds = n // 100
            remainder = n % 100
            result = ones[hundreds] + " trăm"
            if remainder > 0:
                if remainder < 10:
                    result += " lẻ " + ones[remainder]
                else:
                    result += " " + AlwaysOnPipeline._number_to_vietnamese(remainder)
            return result
        else:
            thousands = n // 1000
            remainder = n % 1000
            result = AlwaysOnPipeline._number_to_vietnamese(thousands) + " nghìn"
            if remainder > 0:
                if remainder < 100:
                    result += " không trăm " + \
                              AlwaysOnPipeline._number_to_vietnamese(remainder)
                else:
                    result += " " + AlwaysOnPipeline._number_to_vietnamese(remainder)
            return result
    
    # ──────────────────────────────────────────────
    #  LIFECYCLE MANAGEMENT
    # ──────────────────────────────────────────────
    
    def start(self):
        """Khởi động pipeline."""
        logger.info("=" * 50)
        logger.info("EV-VoiceGuard Starting...")
        logger.info("=" * 50)
        
        self._load_models()
        
        # Khởi chạy threads
        self._producer_thread = threading.Thread(
            target=self.producer_audio_vad_thread,
            name="AudioVAD-Producer",
            daemon=True,
        )
        self._consumer_thread = threading.Thread(
            target=self.consumer_asr_thread,
            name="ASR-TTS-Consumer",
            daemon=True,
        )
        
        self._producer_thread.start()
        self._consumer_thread.start()
        
        logger.info("Pipeline started. Listening...")
    
    def stop(self):
        """Dừng pipeline an toàn (graceful shutdown)."""
        logger.info("Shutting down pipeline...")
        self._running.clear()  # Signal tất cả threads dừng
        
        # Chờ threads kết thúc (timeout 5 giây)
        self._producer_thread.join(timeout=5.0)
        self._consumer_thread.join(timeout=5.0)
        
        # In thống kê cuối
        logger.info(f"Final stats: {self._stats}")
        logger.info("Pipeline stopped.")
    
    @property
    def stats(self) -> dict:
        with self._stats_lock:
            return self._stats.copy()


# ──────────────────────────────────────────────
#  ENTRY POINT
# ──────────────────────────────────────────────

if __name__ == "__main__":
    import signal
    
    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s [%(threadName)s] %(levelname)s: %(message)s",
    )
    
    config = {
        "sample_rate": 16000,
        "chunk_size": 480,          # 30ms @ 16kHz
        "buffer_chunks": 100,       # 3 giây Ring Buffer
        "queue_maxsize": 10,        # Tối đa 10 utterances trong queue
        "vad_threshold": 0.5,
        "max_speech_duration": 30.0,
        "min_silence_duration": 0.5,
        "speech_pad_chunks": 10,    # 300ms padding
    }
    
    pipeline = AlwaysOnPipeline(config)
    
    # Graceful shutdown on Ctrl+C
    def signal_handler(sig, frame):
        pipeline.stop()
    
    signal.signal(signal.SIGINT, signal_handler)
    signal.signal(signal.SIGTERM, signal_handler)
    
    pipeline.start()
    
    # Giữ main thread sống
    try:
        while pipeline._running.is_set():
            time.sleep(1)
    except KeyboardInterrupt:
        pipeline.stop()
```

---

## 7. CÂU HỎI GIẢI TRÌNH (DOCUMENTATION)

### 7.1 Xử lý đa ngôn ngữ (Code-switching)

**Phương án được chọn: Text-normalization bằng Regex/Rules tại tầng phần mềm.**

#### Lý do

1. **Tốc độ CPU:** Regex matching trên chuỗi ngắn (<500 ký tự) mất <1ms trên ARM Cortex-A76. So với can thiệp Lexicon/Tokenizer (cần rebuild embedding matrix, tăng inference time 10-30%), phương án này gần như "miễn phí" về mặt tính toán.

2. **Kích thước model không đổi:** Model TTS giữ nguyên ~74.8M params. Không cần thêm embedding vectors cho từ vựng tiếng Anh → Không vi phạm ràng buộc <100M params.

3. **Dễ bảo trì trong môi trường sản xuất:**
   - Thêm thuật ngữ mới = thêm 1 dòng vào file `.dict`
   - Không cần retrain/finetune model
   - Cập nhật OTA (Over-the-Air) chỉ cần push file text, không cần push model mới (~300MB)

4. **Xử lý fallback mạnh mẽ:** Nếu thuật ngữ không có trong lexicon → Áp dụng heuristic phiên âm tự động (grapheme-to-phoneme rules cho tiếng Anh → tiếng Việt).

#### Nhược điểm chấp nhận được

- Phiên âm thủ công có thể không chuẩn 100% ngữ điệu tiếng Anh gốc.
- Cần effort ban đầu để xây dựng lexicon cho domain xe điện (~200-500 từ).
- Edge case: từ viết tắt trùng từ thông dụng (ví dụ: "CAN" vừa là giao thức, vừa là từ tiếng Anh → cần context-aware rule).

### 7.2 Kiểm soát Ngữ điệu (Prosody Control)

**Phương án được chọn: Kết hợp Post-processing DSP + Conditioning Input (nếu model hỗ trợ).**

#### Chiến lược 2 tầng

```
Tầng 1 (Ưu tiên): Model Conditioning
─────────────────────────────────────
Nếu Valtec-TTS hỗ trợ duration/pitch conditioning:
  → Điều chỉnh trực tiếp input tensors
  → Inference time KHÔNG TĂNG (chỉ thay giá trị input)
  → Chất lượng tốt nhất

Tầng 2 (Fallback): Post-processing DSP
─────────────────────────────────────
Nếu model không hỗ trợ conditioning:
  → Dùng librosa/scipy để time-stretch + pitch-shift
  → Thêm ~5-10ms processing (negligible)
  → Chất lượng giảm nhẹ khi stretch > 1.3x
```

**Lý do:** Cả hai phương án đều không tăng inference time đáng kể. Tầng 1 cho chất lượng tốt nhất; Tầng 2 đảm bảo tính tương thích với mọi model TTS.

### 7.3 Quản lý Hàng đợi & Backpressure

**Phương án được chọn: Drop-oldest + VAD Timeout.**

#### Cơ chế hoạt động

```
┌─────────────────────────────────────────────────────┐
│              BACKPRESSURE STRATEGY                    │
│                                                      │
│  1. Queue maxsize = 10 utterances                    │
│     → Giới hạn bộ nhớ queue ≤ ~15MB (10 × 30s max)  │
│                                                      │
│  2. Khi queue ĐẦY và Producer muốn push:             │
│     → Drop item CŨ NHẤT trong queue                  │
│     → Push item mới vào                              │
│     → Log cảnh báo + tăng drop_counter               │
│                                                      │
│  3. VAD Timeout = 30 giây                            │
│     → Nếu speech liên tục > 30s → Force flush        │
│     → Ngăn speech_buffer vô hạn                      │
│     → Xử lý case: môi trường quá ồn liên tục        │
│                                                      │
│  4. VAD min_silence_duration = 500ms                 │
│     → Phân tách utterances tự nhiên                  │
│     → Mỗi utterance ~2-10 giây (typical)             │
└─────────────────────────────────────────────────────┘
```

#### Tác dụng phụ lên trải nghiệm người dùng

| Tình huống | Hành vi hệ thống | Ảnh hưởng UX |
|------------|-------------------|--------------|
| Nói bình thường (<10s) | Queue xử lý kịp | ✅ Không ảnh hưởng |
| Nói dài không ngắt (>30s) | Force flush mỗi 30s | ⚠️ Câu bị cắt đoạn, nhưng vẫn xử lý |
| Ồn liên tục (VAD false positive) | Queue đầy → Drop oldest | ⚠️ Mất command cũ, nhưng hệ thống không treo |
| Burst nhiều người nói | Drop oldest frames | ⚠️ Chỉ xử lý câu gần nhất |

**Trade-off được chấp nhận:** Mất dữ liệu cũ tốt hơn hệ thống bị treo. Trong ngữ cảnh xe hơi, command gần nhất luôn có ý nghĩa hơn command cũ.

---

## 8. PHỤ LỤC

### A. Công thức tính Memory Budget

```
Ring Buffer:     480 × 2 × 100              =    96,000 bytes ≈   94 KB
Queue (max):     16000 × 2 × 30 × 10        = 9,600,000 bytes ≈  9.2 MB
VAD Model:       Silero ONNX                 ≈ 2,000,000 bytes ≈  1.9 MB
ASR Model:       SenseVoice INT8             ≈63,000,000 bytes ≈ 60.0 MB
TTS Model:       Valtec-TTS                  ≈80,000,000 bytes ≈ 76.3 MB
Lexicon + Rules: Text files                  ≈    50,000 bytes ≈   49 KB
──────────────────────────────────────────────────────────────────────────
TOTAL ESTIMATED:                              ≈              ~148 MB
Available RAM (4GB model):                    ≈            3,500 MB
Headroom:                                     ≈            3,352 MB ✅
```
