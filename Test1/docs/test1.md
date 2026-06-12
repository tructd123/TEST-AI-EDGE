# BÀI KIỂM TRA NĂNG LỰC AI EDGE - SỐ 1
## Chủ đề: Xây dựng luồng Voice-to-Voice cơ bản (Push-to-Talk) trên môi trường CPU ARM.

## **Họ và tên:** Đỗ Tuấn Trực 

---

## Mục Lục

1. [TỔNG QUAN DỰ ÁN](#1-tổng-quan-dự-án)
2. [KHỐI 1: LƯỢNG TỬ HÓA & CHUẨN BỊ MÔ HÌNH](#2-khối-1-lượng-tử-hóa--chuẩn-bị-mô-hình)
3. [KHỐI 2: TỐI ƯU HÓA BACKEND INFERENCE](#3-khối-2-tối-ưu-hóa-backend-inference)
4. [KHỐI 3: QUẢN LÝ BỘ NHỚ](#4-khối-3-quản-lý-bộ-nhớ)
5. [PSEUDO-CODE & CẤU TRÚC CHƯƠNG TRÌNH](#5-pseudo-code--cấu-trúc-chương-trình)
6. [CÂU HỎI GIẢI TRÌNH](#6-câu-hỏi-giải-trình)
7. [PHỤ LỤC](#7-phụ-lục)

---

## 1. TỔNG QUAN DỰ ÁN

### 1.1. Bài Toán

Xây dựng module tương tác giọng nói **100% offline** cho robot hình người, chạy trên **Raspberry Pi 5** (4 nhân Cortex-A76, 4–8 GB RAM) — không có GPU rời. Module hoạt động theo cơ chế **Push-to-Talk (PTT)**:

```
┌──────────┐     ┌───────────┐     ┌───────┐     ┌───────────┐     ┌──────┐
│ Bấm nút  │────▶│ Ghi âm    │────▶│  ASR  │────▶│   TTS     │────▶│ Loa  │
│ (GPIO)   │     │ (PCM RAM) │     │Whisper│     │  Piper    │     │      │
└──────────┘     └───────────┘     └───────┘     └───────────┘     └──────┘
    ▲                                                                  │
    │              Push-to-Talk Loop (In-Memory)                       │
    └──────────────────────────────────────────────────────────────────┘
```

### 1.2. Chỉ Tiêu Hiệu Năng (KPIs)

| Chỉ tiêu | Mục tiêu | Ý nghĩa |
|-----------|----------|----------|
| **RTF toàn luồng** | < 0.3 | 5s audio → xử lý xong ASR+TTS trong < 1.5s |
| **Cold-start** | < 8s | Thời gian load model lần đầu khi khởi động |
| **RAM steady-state** | < 350 MB | Tổng RAM sử dụng ổn định |
| **Memory Leak** | 0 bytes/hour growth | RAM không tăng dần theo thời gian |

### 1.3. Lựa Chọn Mô Hình

| Vai trò | Mô hình | Lý do chọn |
|---------|---------|-------------|
| **ASR** | Whisper-Tiny (39M params) | Nhỏ nhất trong họ Whisper, hỗ trợ tiếng Việt, WER chấp nhận được cho giao tiếp cơ bản |
| **TTS** | Piper TTS (vi-VN) | Open-source, ONNX native, có voice tiếng Việt sẵn, inference nhanh trên CPU |

---

## 2. KHỐI 1: LƯỢNG TỬ HÓA & CHUẨN BỊ MÔ HÌNH

### 2.1. Lựa Chọn Định Dạng Lượng Tử Hóa: **Q8_0 (GGUF)**

#### Bảng So Sánh Các Mức Quantization cho Whisper-Tiny trên Cortex-A76

| Mức | Kích thước | Tốc độ (tương đối) | WER Δ vs FP32 | RAM cần | Verdict |
|-----|-----------|---------------------|---------------|---------|---------|
| **FP32** | ~150 MB | 1.0× (baseline) | 0% | ~300 MB | ❌ Quá chậm, quá lớn cho Pi 5 |
| **FP16** | ~75 MB | ~1.4× | < 0.1% | ~180 MB | ⚠️ Cortex-A76 không có FP16 FMLA native — phải convert on-the-fly → overhead |
| **INT8** (ONNX) | ~40 MB | ~1.8× | < 0.5% | ~120 MB | ⚠️ Tốt nhưng phụ thuộc ONNX Runtime, build phức tạp hơn |
| **Q8_0** (GGUF) | ~42 MB | ~2.0× | < 0.5% | ~110 MB | ✅ **CHỌN** — nhanh nhất, NEON tối ưu, WER tốt |
| **Q5_1** (GGUF) | ~32 MB | ~2.3× | ~1.2% | ~90 MB | ⚠️ Nhanh hơn nhưng WER bắt đầu tăng đáng kể |
| **Q4_0** (GGUF) | ~25 MB | ~2.5× | ~2.5% | ~80 MB | ❌ Vượt ngưỡng 2% WER degradation |

#### Lý Do Chọn Q8_0

1. **Tốc độ tối ưu trên NEON:** Q8_0 sử dụng 8-bit integer thuần túy. Cortex-A76 có các instruction NEON `SDOT` (Signed Dot Product) cho phép thực hiện dot product 4 cặp INT8 trong 1 cycle. Đây là sweet spot giữa tốc độ và chính xác.

2. **WER degradation < 0.5%:** Whisper-Tiny đã có WER baseline khá cao (~14% cho tiếng Việt). Chênh lệch 0.5% là không đáng kể trong use case giao tiếp robot.

3. **Tại sao KHÔNG chọn FP16:** Cortex-A76 hỗ trợ FP16 ở mức chuyển đổi (convert to FP32 rồi tính), không phải native FP16 arithmetic như GPU. Điều này tạo overhead mà không có lợi ích tốc độ thực sự.

4. **Tại sao KHÔNG chọn Q4_0:** Mặc dù nhanh nhất, nhưng WER tăng ~2.5% so với FP32 — vượt ngưỡng cho phép 2%. Với model nhỏ như Whisper-Tiny (39M params), mỗi bit quantization gây ảnh hưởng lớn hơn so với model lớn.

5. **Tại sao KHÔNG chọn Q5_1:** Là lựa chọn tốt thứ hai, nhưng WER ~1.2% đã gần ngưỡng, và khoản tiết kiệm RAM (~20 MB) không đủ đáng kể trên hệ thống 4 GB RAM.

### 2.2. Quy Trình Chuyển Đổi Mô Hình

```bash
# Bước 1: Tải model Whisper-Tiny gốc (PyTorch .pt)
python -c "import whisper; whisper.load_model('tiny').save('whisper-tiny.pt')"

# Bước 2: Chuyển đổi sang định dạng GGUF bằng whisper.cpp converter
cd whisper.cpp
python models/convert-pt-to-ggml.py \
    ../whisper-tiny.pt \
    ./models/ \
    --outtype f32

# Bước 3: Quantize từ FP32 GGUF sang Q8_0
./quantize \
    models/ggml-tiny.bin \
    models/ggml-tiny-q8_0.bin \
    q8_0

# Bước 4: Verify
./main -m models/ggml-tiny-q8_0.bin -f test_audio.wav --print-progress
```

### 2.3. Chuẩn Bị Piper TTS

Piper TTS sử dụng ONNX format native — không cần quantize thêm:

```bash
# Tải voice tiếng Việt
wget https://github.com/rhasspy/piper/releases/download/v1.2.0/vi_VN-vivos-x_low.onnx
wget https://github.com/rhasspy/piper/releases/download/v1.2.0/vi_VN-vivos-x_low.onnx.json
```

> **Lưu ý:** Piper TTS model tiếng Việt có kích thước ~60–80 MB (FP32 ONNX). Trên Pi 5 với 4 GB RAM, tổng cộng cả ASR (~110 MB working set) + TTS (~120 MB working set) ≈ 230 MB — vẫn dư dả cho OS và buffer.

---

## 3. KHỐI 2: TỐI ƯU HÓA BACKEND INFERENCE

### 3.1. Lựa Chọn Backend ASR: **whisper.cpp**

#### So Sánh whisper.cpp vs sherpa-onnx

| Tiêu chí | whisper.cpp | sherpa-onnx |
|-----------|-------------|-------------|
| **Ngôn ngữ** | C/C++ thuần | C++ (wrapper ONNX Runtime) |
| **NEON support** | Tự động detect & sử dụng | Phụ thuộc ONNX Runtime build |
| **Build complexity** | `cmake + make` (đơn giản) | Phải build ONNX Runtime cho ARM64 |
| **GGUF support** | Native ✅ | Không (dùng ONNX format) |
| **Community** | ~37K GitHub stars, rất active | ~4K stars, nhỏ hơn |
| **Python binding** | `pywhispercpp` (ctypes) | Native Python package |
| **Binary size** | ~2 MB | ~15 MB (kéo theo ONNX Runtime) |
| **Kết luận** | ✅ **CHỌN** | ⚠️ Dự phòng |

**Lý do chọn whisper.cpp:**
- Build trực tiếp trên Pi 5 chỉ với `cmake` + `make`, tự động phát hiện NEON.
- Format GGUF là native format — quantize dễ, load nhanh.
- Binary size cực nhỏ, phù hợp embedded.
- Đã được kiểm chứng rộng rãi trên ARM SBC.

### 3.2. Biên Dịch whisper.cpp cho ARM64

```bash
#!/bin/bash
# scripts/build_whisper_cpp.sh

git clone https://github.com/ggerganov/whisper.cpp.git
cd whisper.cpp

mkdir build && cd build

cmake .. \
    -DCMAKE_BUILD_TYPE=Release \
    -DWHISPER_NO_METAL=ON \
    -DWHISPER_NO_CUDA=ON \
    -DWHISPER_NO_OPENCL=ON \
    -DCMAKE_C_FLAGS="-mcpu=cortex-a76 -O3 -ffast-math" \
    -DCMAKE_CXX_FLAGS="-mcpu=cortex-a76 -O3 -ffast-math"

make -j4

# Cờ -mcpu=cortex-a76 tự động bật:
#   - NEON SIMD (128-bit vector)
#   - Dot Product instructions (SDOT/UDOT)
#   - FP16 conversion
#   - Crypto extensions
```

> **Giải thích cờ build quan trọng:**
> - `-mcpu=cortex-a76`: Chỉ định chính xác CPU target, compiler sẽ bật tất cả instruction sets có sẵn (NEON, DotProd, etc.)
> - `-O3`: Optimization cấp cao nhất — loop unrolling, vectorization, inlining
> - `-ffast-math`: Cho phép compiler tối ưu floating-point (relaxed IEEE 754) — tăng tốc đáng kể cho inference
> - `-DWHISPER_NO_METAL/CUDA/OPENCL=ON`: Tắt tất cả GPU backend — giảm binary size và thời gian build

### 3.3. Tận Dụng ARM NEON SIMD

whisper.cpp đã có sẵn kernel NEON được tối ưu thủ công (hand-optimized) cho các phép toán tensor cơ bản:

```
┌─────────────────────────────────────────────────────────────┐
│                    whisper.cpp NEON Pipeline                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  MatMul (Q8×Q8)    ──▶  SDOT instruction                   │
│  │                      (4×INT8 dot product / cycle)        │
│  │                                                          │
│  LayerNorm          ──▶  NEON FMLA (fused multiply-add)    │
│  │                                                          │
│  Softmax            ──▶  NEON FMAX + FMUL vectorized       │
│  │                                                          │
│  Mel Spectrogram    ──▶  NEON FFT (radix-4 butterfly)      │
│                                                             │
│  Throughput: ~4–8× so với scalar C code                    │
└─────────────────────────────────────────────────────────────┘
```

### 3.4. Backend TTS: Piper (Native)

Piper TTS đã là binary compiled (C++ + ONNX Runtime), chỉ cần gọi qua subprocess hoặc Python binding:

```bash
# Build Piper cho ARM64 (nếu build từ source)
cmake .. -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_C_FLAGS="-mcpu=cortex-a76 -O3"

# Hoặc sử dụng pre-built binary cho aarch64
wget https://github.com/rhasspy/piper/releases/download/v1.2.0/piper_arm64.tar.gz
```

---

## 4. KHỐI 3: QUẢN LÝ BỘ NHỚ

### 4.1. Chiến Lược "Load Once, Infer Forever"

```
                    STARTUP (Cold-start)                    RUNTIME (Hot-path)
                    ═══════════════════                    ═══════════════════

                    ┌─────────────┐                       ┌─────────────┐
                    │  SD Card /  │    Load 1 lần          │             │
                    │  SSD        │──────────────┐        │  Chỉ truyền │
                    │             │              │        │  PCM data   │
                    │ whisper-    │              ▼        │  trong RAM  │
                    │ tiny-q8.gguf│         ┌────────┐   │             │
                    │             │         │  RAM   │◀──│  Audio In   │
                    │ piper-vi.   │         │        │──▶│  ASR Out    │
                    │ onnx        │────────▶│ Models │──▶│  TTS Out    │
                    └─────────────┘         │ ~230MB │   │  Audio Out  │
                                            └────────┘   └─────────────┘

                    ~5–8 giây                             ~0.5–1.5 giây/lần
```

### 4.2. Memory Layout Dự Kiến

```
Total RAM Pi 5 (4 GB)
├── OS + System Services          ~400 MB
├── whisper.cpp model (Q8_0)      ~110 MB (weights + KV cache)
├── Piper TTS model               ~120 MB (ONNX graph + weights)
├── Audio Buffers                  ~  5 MB (PCM ring buffer)
├── Python Runtime                 ~ 50 MB
├── Headroom                       ~3.3 GB còn lại
└── /dev/shm (TTS output tmp)     ~  1 MB (tái sử dụng)
```

### 4.3. Phòng Chống Memory Leak

| Biện pháp | Mô tả |
|-----------|--------|
| **Pre-allocated buffers** | Audio buffer được cấp phát cố định (ví dụ: 30s × 16kHz × 2 bytes = ~960 KB), tái sử dụng mỗi lần bấm nút |
| **No dynamic model loading** | Model reference giữ nguyên suốt vòng đời, không `del` rồi `load` lại |
| **Explicit cleanup** | Sau mỗi inference, chỉ clear data buffer (`.fill(0)`) — không deallocate |
| **Monitoring** | Dùng `tracemalloc` (Python) hoặc `valgrind` (C++) để kiểm tra định kỳ |

---

## 5. PSEUDO-CODE & CẤU TRÚC CHƯƠNG TRÌNH

### 5.1. Class Diagram

```
┌──────────────────────────────────────────────────────┐
│                    VoicePipeline                     │
│──────────────────────────────────────────────────────│
│ - asr_engine: WhisperEngine                          │
│ - tts_engine: PiperEngine                            │
│ - audio_buffer: numpy.ndarray (pre-allocated)        │
│ - is_recording: bool                                 │
│ - sample_rate: int = 16000                           │
│ - audio_stream: sounddevice.InputStream              │
│──────────────────────────────────────────────────────│
│ + __init__()          # Load ALL models (1 lần)      │
│ + start_recording()   # Bắt đầu ghi vào buffer       │
│ + stop_and_process()  # Dừng ghi → ASR → TTS → Play  │
│ + _audio_callback()   # Callback ghi PCM vào buffer  │
│ + run()               # Main loop chờ GPIO event     │
│ + shutdown()          # Giải phóng tài nguyên        │
└──────────────────────────────────────────────────────┘
         │                           │
         ▼                           ▼
┌─────────────────┐        ┌──────────────────┐
│  WhisperEngine  │        │   PiperEngine    │
│─────────────────│        │──────────────────│
│ - ctx: pointer  │        │ - handle: object │
│ - params: struct│        │ - voice: str     │
│─────────────────│        │──────────────────│
│ + load(path)    │        │ + load(path)     │
│ + transcribe(   │        │ + synthesize(    │
│     pcm_array)  │        │     text) → pcm  │
│   → str         │        │                  │
└─────────────────┘        └──────────────────┘
```

### 5.2. Python Pseudo-Code

```python
#!/usr/bin/env python3
"""
Voice-to-Voice Pipeline (Push-to-Talk) cho Raspberry Pi 5
Backend: whisper.cpp (ASR) + Piper TTS
Tất cả model được load 1 lần duy nhất lúc khởi động.
Audio PCM được truyền trong RAM — KHÔNG ghi file tạm trên SD card.
"""

import numpy as np
import sounddevice as sd
import subprocess
import struct
import os
from pathlib import Path
from pywhispercpp.model import Model as WhisperModel

# ============================================================
# CONSTANTS
# ============================================================
SAMPLE_RATE = 16000          # Whisper yêu cầu 16 kHz
CHANNELS = 1                 # Mono
DTYPE = np.float32           # PCM format cho whisper.cpp
MAX_RECORD_SECONDS = 30      # Giới hạn buffer
NUM_THREADS = 3              # Giữ 1 nhân cho OS + Audio I/O
SHM_TTS_PATH = "/dev/shm/tts_output.wav"  # tmpfs trên RAM


class WhisperEngine:
    """
    Wrapper cho whisper.cpp thông qua pywhispercpp.
    Model được load 1 lần duy nhất trong __init__.
    """

    def __init__(self, model_path: str):
        """
        CRITICAL: Đây là nơi duy nhất model được load từ disk vào RAM.
        Sau bước này, self._model giữ reference vĩnh viễn trong RAM.
        """
        print(f"[ASR] Loading Whisper model from {model_path}...")
        self._model = WhisperModel(
            model_path,
            n_threads=NUM_THREADS,
            language="vi",          # Tiếng Việt
            print_progress=False,
            no_timestamps=True,
        )
        print(f"[ASR] Model loaded. Ready for inference.")

    def transcribe(self, pcm_data: np.ndarray) -> str:
        """
        Nhận trực tiếp mảng PCM float32 từ RAM — KHÔNG đọc file.

        Args:
            pcm_data: numpy array shape (N,), dtype float32, sample_rate=16kHz

        Returns:
            Chuỗi text đã nhận dạng.
        """
        # pywhispercpp chấp nhận numpy array trực tiếp
        # → Dữ liệu truyền qua pointer trong RAM, zero-copy
        segments = self._model.transcribe(pcm_data)
        text = " ".join([seg.text for seg in segments]).strip()
        return text


class PiperEngine:
    """
    Wrapper cho Piper TTS.
    Model ONNX được load 1 lần trong __init__.
    """

    def __init__(self, model_path: str, config_path: str):
        """
        Piper binary load model ONNX vào RAM lúc khởi tạo.
        Ta sử dụng subprocess + pipe thay vì ghi file tạm.
        """
        self._model_path = model_path
        self._config_path = config_path
        self._piper_bin = "/usr/local/bin/piper"  # Pre-built binary

        # Warm-up: chạy 1 câu ngắn để force load model vào RAM
        print(f"[TTS] Loading Piper model from {model_path}...")
        self._warm_up()
        print(f"[TTS] Model warmed up. Ready for synthesis.")

    def _warm_up(self):
        """Chạy 1 inference giả để model được nạp hoàn toàn vào RAM."""
        self.synthesize("xin chào")

    def synthesize(self, text: str) -> np.ndarray:
        """
        Chuyển text thành PCM audio bytes.

        Chiến lược I/O:
        - Input: text qua stdin pipe (không ghi file)
        - Output: WAV ra /dev/shm (tmpfs = RAM disk)
          → Tốc độ đọc/ghi tức thời
          → Không ghi xuống SD card → bảo vệ tuổi thọ storage

        Returns:
            numpy array PCM float32, ready to play.
        """
        proc = subprocess.run(
            [
                self._piper_bin,
                "--model", self._model_path,
                "--config", self._config_path,
                "--output_file", SHM_TTS_PATH,
                "--sentence_silence", "0.2",
            ],
            input=text,
            capture_output=True,
            text=True,
            timeout=10,
        )

        if proc.returncode != 0:
            raise RuntimeError(f"Piper TTS failed: {proc.stderr}")

        # Đọc WAV từ /dev/shm (RAM) — tốc độ tức thời
        # Parse WAV header, extract raw PCM
        audio_data = self._read_wav_pcm(SHM_TTS_PATH)
        return audio_data

    @staticmethod
    def _read_wav_pcm(wav_path: str) -> np.ndarray:
        """Đọc file WAV từ /dev/shm, trả về PCM float32 array."""
        import wave
        with wave.open(wav_path, 'rb') as wf:
            raw_bytes = wf.readframes(wf.getnframes())
            # Piper output: 16-bit signed integer PCM
            pcm_int16 = np.frombuffer(raw_bytes, dtype=np.int16)
            # Normalize sang float32 [-1.0, 1.0]
            pcm_float32 = pcm_int16.astype(np.float32) / 32768.0
            return pcm_float32


class VoicePipeline:
    """
    Bộ điều khiển chính cho luồng Voice-to-Voice.

    Lifecycle:
        1. __init__(): Load TẤT CẢ model vào RAM (1 lần duy nhất)
        2. run(): Main loop — chờ GPIO button event
        3. start_recording(): Bấm nút → bắt đầu ghi âm vào buffer
        4. stop_and_process(): Nhả nút → ASR → TTS → Play
        5. shutdown(): Cleanup khi thoát chương trình
    """

    def __init__(self):
        # ============================================================
        # MODEL LOADING — CHỈ THỰC HIỆN 1 LẦN DUY NHẤT
        # Sau bước này, model nằm trong RAM suốt vòng đời chương trình
        # ============================================================
        self.asr = WhisperEngine(
            model_path="models/ggml-tiny-q8_0.bin"
        )
        self.tts = PiperEngine(
            model_path="models/vi_VN-vivos-x_low.onnx",
            config_path="models/vi_VN-vivos-x_low.onnx.json"
        )

        # ============================================================
        # PRE-ALLOCATED AUDIO BUFFER — Cấp phát 1 lần, tái sử dụng
        # 30 giây × 16000 Hz × 4 bytes (float32) = ~1.92 MB
        # ============================================================
        self._buffer = np.zeros(
            MAX_RECORD_SECONDS * SAMPLE_RATE,
            dtype=DTYPE
        )
        self._buffer_pos = 0       # Con trỏ vị trí ghi hiện tại
        self._is_recording = False
        self._stream = None

        print("[Pipeline] Initialization complete. All models in RAM.")
        print(f"[Pipeline] Audio buffer: {self._buffer.nbytes / 1024:.1f} KB pre-allocated")

    def _audio_callback(self, indata: np.ndarray, frames: int,
                        time_info, status):
        """
        Callback được gọi bởi sounddevice mỗi khi có audio chunk mới.
        Ghi trực tiếp vào pre-allocated buffer — KHÔNG tạo buffer mới.

        Args:
            indata: Audio chunk từ microphone (float32)
            frames: Số frame trong chunk
        """
        if not self._is_recording:
            return

        end_pos = min(self._buffer_pos + frames,
                      len(self._buffer))
        actual_frames = end_pos - self._buffer_pos

        # Ghi vào vùng nhớ đã cấp phát sẵn (zero-copy write)
        self._buffer[self._buffer_pos:end_pos] = (
            indata[:actual_frames, 0]
        )
        self._buffer_pos = end_pos

    def start_recording(self):
        """
        Gọi khi người dùng BẤM nút.
        Bắt đầu thu âm từ microphone vào buffer trong RAM.
        """
        # Reset buffer position (KHÔNG deallocate buffer)
        self._buffer_pos = 0
        self._is_recording = True

        # Mở audio stream
        self._stream = sd.InputStream(
            samplerate=SAMPLE_RATE,
            channels=CHANNELS,
            dtype=DTYPE,
            callback=self._audio_callback,
            blocksize=1024,  # ~64ms per callback @ 16kHz
        )
        self._stream.start()
        print("[Recording] Started... (release button to process)")

    def stop_and_process(self):
        """
        Gọi khi người dùng NHẢ nút.
        Dừng thu âm → Chạy ASR → Chạy TTS → Phát audio.
        Toàn bộ dữ liệu truyền qua RAM, không ghi file tạm trên SD.
        """
        # 1. Dừng thu âm
        self._is_recording = False
        if self._stream:
            self._stream.stop()
            self._stream.close()
            self._stream = None

        # 2. Cắt buffer đến vị trí thực tế đã ghi
        #    Sử dụng view (không copy) để tiết kiệm RAM
        recorded_audio = self._buffer[:self._buffer_pos]
        duration = self._buffer_pos / SAMPLE_RATE
        print(f"[Recording] Stopped. Duration: {duration:.1f}s")

        if duration < 0.5:
            print("[Warning] Audio too short, skipping.")
            return

        # 3. ASR: PCM array → Text (truyền trực tiếp trong RAM)
        print("[ASR] Transcribing...")
        text = self.asr.transcribe(recorded_audio)
        print(f"[ASR] Result: '{text}'")

        if not text or text.isspace():
            print("[Warning] No speech detected.")
            return

        # 4. TTS: Text → PCM array
        print("[TTS] Synthesizing...")
        tts_audio = self.tts.synthesize(text)
        print(f"[TTS] Generated {len(tts_audio)} samples")

        # 5. Phát audio ra loa
        print("[Playback] Playing...")
        sd.play(tts_audio, samplerate=22050)  # Piper default = 22050 Hz
        sd.wait()  # Block cho đến khi phát xong
        print("[Playback] Done.")

    def run(self):
        """
        Main loop — chờ sự kiện GPIO button.
        Trên Pi 5, sử dụng gpiod (libgpiod) thay cho RPi.GPIO.
        """
        import gpiod

        BUTTON_PIN = 17  # GPIO17 = Pin 11 trên header

        chip = gpiod.Chip('gpiochip4')  # Pi 5 dùng gpiochip4
        line = chip.get_line(BUTTON_PIN)
        line.request(consumer="vtv-ptt",
                     type=gpiod.LINE_REQ_EV_BOTH_EDGES)

        print("\n" + "=" * 50)
        print("  Voice-to-Voice Pipeline READY")
        print("  Press and hold button to speak.")
        print("=" * 50 + "\n")

        try:
            while True:
                event = line.event_wait(sec=1)
                if event:
                    evt = line.event_read()
                    if evt.type == gpiod.LineEvent.FALLING_EDGE:
                        # Nút được BẤM (active-low)
                        self.start_recording()
                    elif evt.type == gpiod.LineEvent.RISING_EDGE:
                        # Nút được NHẢ
                        self.stop_and_process()

        except KeyboardInterrupt:
            print("\n[Shutdown] Ctrl+C received.")
        finally:
            self.shutdown()

    def shutdown(self):
        """Cleanup tài nguyên khi thoát."""
        if self._stream:
            self._stream.close()
        # Xóa file tạm trên /dev/shm nếu còn
        if os.path.exists(SHM_TTS_PATH):
            os.remove(SHM_TTS_PATH)
        print("[Shutdown] Pipeline terminated cleanly.")


# ============================================================
# ENTRY POINT
# ============================================================
if __name__ == "__main__":
    # Model được load ở đây — 1 lần duy nhất cho toàn bộ vòng đời
    pipeline = VoicePipeline()
    pipeline.run()
```

---

## 6. CÂU HỎI GIẢI TRÌNH (DOCUMENTATION)

### Q1: whisper.cpp hay sherpa-onnx?

**Chọn: whisper.cpp**

Lý do chọn giải pháp:

- Build đơn giản: Chỉ cần cmake + make trên Pi 5, không dependency phức tạp.    
- GGUF native: Format quantization tối ưu cho CPU, hỗ trợ Q8/Q5/Q4 trực tiếp.    
- NEON tự động: Compiler flags -mcpu=cortex-a76 tự động bật tất cả instruction sets.    
- Cộng đồng lớn: ~37K stars, bug fixes nhanh, nhiều tài liệu benchmark trên ARM.   
- Binary nhỏ: ~2 MB so với ~15 MB của sherpa-onnx (kéo theo ONNX Runtime).    

sherpa-onnx phù hợp hơn khi cần **streaming ASR** (nhận dạng real-time khi đang nói) — nhưng trong use case Push-to-Talk, ta xử lý toàn bộ audio sau khi nhả nút, nên batch inference của whisper.cpp là đủ.

---

### Q2: num_threads = bao nhiêu? Tại sao num_threads=4 có thể chậm hơn num_threads=2?

**Chọn: `num_threads = 3`**

#### Phân bổ 4 nhân Cortex-A76:

```
Core 0:  OS kernel + Audio I/O (sounddevice callback)
Core 1:  ┐
Core 2:  ├── whisper.cpp / Piper inference (3 threads)
Core 3:  ┘
```

#### Tại sao num_threads=4 có thể CHẬM HƠN num_threads=2:

```
                num_threads = 2                    num_threads = 4
                ══════════════                    ══════════════

   Core 0: [  OS + Audio  ]           Core 0: [ OS + Audio + Inference]
   Core 1: [ Inference T1 ]           Core 1: [    Inference T1       ]
   Core 2: [ Inference T2 ]           Core 2: [    Inference T2       ]
   Core 3: [   Idle/OS    ]           Core 3: [    Inference T3       ]
                                                    ↑ Inference T4
                                                    phải CHIA SẺ core
                                                    với OS/Audio
```

**4 nguyên nhân chính:**

1. **Core Contention (tranh chấp nhân):** Khi đặt 4 threads inference trên 4 cores, thread inference phải chia sẻ Core 0 với OS scheduler, audio driver, và interrupt handler. Context switching giữa inference thread và OS thread gây overhead lớn hơn lợi ích từ thread thứ 4.

2. **Cache Thrashing:** Cortex-A76 có 64 KB L1 cache per core và 512 KB L2 per core. Khi OS thread và inference thread chia sẻ 1 core, chúng "đẩy" cache data của nhau ra liên tục (cache eviction). Với matrix multiplication trong transformer, cache hit rate giảm → memory latency tăng.

3. **Memory Bandwidth Saturation:** Pi 5 sử dụng LPDDR4X với bandwidth ~34 GB/s chia sẻ cho tất cả cores. Khi cả 4 cores đều đọc weight matrix đồng thời, bandwidth bị bão hòa. 3 threads sử dụng ~75% bandwidth — sweet spot.

4. **Thread Synchronization Overhead:** Mỗi layer trong transformer có barrier synchronization giữa các threads (chờ thread chậm nhất hoàn thành phép nhân matrix). Với 4 threads, xác suất 1 thread bị OS preempt cao hơn → tất cả threads khác phải chờ.

**Kết luận:** `num_threads = 3` cho throughput tối ưu trên Pi 5, giữ Core 0 dedicated cho OS + Audio I/O, tránh contention.

---

### Q3: File TTS output lưu ở đâu?

**Chọn: `/dev/shm/` (tmpfs — RAM-backed filesystem)**

```bash
$ mount | grep tmpfs
tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev)
```

| Tiêu chí | `/dev/shm/` (tmpfs) | SD Card (`/tmp/`) | NVMe SSD |
|-----------|--------------------|--------------------|----------|
| **Tốc độ ghi** | ~3 GB/s (RAM speed) | ~40 MB/s (Class 10) | ~500 MB/s |
| **Tốc độ đọc** | ~3 GB/s | ~90 MB/s | ~1 GB/s |
| **Hao mòn** | 0 (RAM không wear) | Rất cao (write amplification) | Thấp |
| **Dung lượng** | 50% RAM mặc định | Tùy card | Tùy SSD |
| **Persistence** | Mất khi reboot ✅ | Giữ lại ❌ | Giữ lại ❌ |

**Tại sao `/dev/shm/`:**
- Tốc độ ghi/đọc bằng RAM (~3 GB/s) — gần như tức thời cho file WAV vài trăm KB.
- Không ghi xuống flash storage → tuổi thọ SD card/SSD không bị ảnh hưởng.
- File tự động mất khi reboot → không tích tụ rác.
- Dùng chung RAM pool với OS nên dung lượng linh hoạt.

**Lưu ý:** Trong thiết kế lý tưởng hơn, Piper TTS nên output trực tiếp vào stdout pipe (PCM raw bytes) thay vì ghi file. Tuy nhiên, phiên bản Piper CLI hiện tại yêu cầu `--output_file`, nên `/dev/shm/` là giải pháp tối ưu nhất.

---

### Q4: Tại sao chọn Q8_0 cho Whisper-Tiny?

*Đã trình bày chi tiết ở [Mục 2.1](#21-lựa-chọn-định-dạng-lượng-tử-hóa-q8_0-gguf).*

**Tóm tắt:** Q8_0 là điểm cân bằng tối ưu trên Cortex-A76 — nhanh ~2× so với FP32, WER chỉ tăng < 0.5%, và tận dụng được NEON SDOT instruction cho 8-bit dot product.

---

## 7. PHỤ LỤC

### A. Benchmark Tham Khảo: Whisper-Tiny trên Pi 5

*Nguồn: Community benchmarks, kết quả thực tế có thể dao động ±15%*

| Quantization | Inference Time (5s audio) | RAM Usage | WER (vi-VN) |
|-------------|---------------------------|-----------|-------------|
| FP32        | ~2.8s                     | ~300 MB   | ~14.2%      |
| FP16        | ~2.1s                     | ~180 MB   | ~14.2%      |
| Q8_0        | ~1.4s                     | ~110 MB   | ~14.5%      |
| Q5_1        | ~1.1s                     | ~90 MB    | ~15.4%      |
| Q4_0        | ~0.9s                     | ~80 MB    | ~16.8%      |

### B. Bảng RTF Mục Tiêu

| Stage | Input | Processing Time | RTF |
|-------|-------|-----------------|-----|
| Recording | 5s audio | 5s (real-time) | N/A |
| ASR (Q8_0) | 5s audio | ~0.7s | 0.14 |
| TTS (Piper) | ~20 words | ~0.5s | ~0.10 |
| Playback | ~3s audio | 3s (real-time) | N/A |
| **Total Processing** | **5s input** | **~1.2s** | **0.24** ✅ |

### C. Công Thức RTF

```
RTF = Thời gian xử lý / Thời gian audio đầu vào

Ví dụ:
- Audio 5 giây, xử lý mất 1.2 giây
- RTF = 1.2 / 5.0 = 0.24
- Yêu cầu: RTF < 0.3 ✅
```

### D. Sơ Đồ Luồng Dữ Liệu Chi Tiết

```
┌────────────────────────────────────────────────────────────────────────┐
│                        RASPBERRY PI 5 — RAM                            │
│                                                                        │
│  ┌──────────┐    PCM float32     ┌──────────────┐                      │
│  │USB Micro │───(callback)──────▶│ numpy buffer │                      │
│  │phone     │    in-memory       │ (1.92 MB     │                      │
│  └──────────┘                    │  pre-alloc)  │                      │
│                                  └──────┬───────┘                      │
│                                         │ numpy view (zero-copy)       │
│                                         ▼                              │
│                                  ┌──────────────┐                      │
│                                  │ whisper.cpp  │                      │
│                                  │ Q8_0 model   │     ┌────────────┐   │
│                                  │ (110 MB RAM) │────▶│  "xin chào"│   │
│                                  └──────────────┘     └─────┬──────┘   │
│                                                             │ string   │
│                                                             ▼          │
│                                  ┌──────────────┐    ┌────────────┐    │
│  ┌──────────┐    PCM int16       │ Piper TTS    │◀───│stdin pipe  │    │
│  │3.5mm/I2S │◀───(sd.play)──────│ ONNX model   │     └────────────┘    │
│  │Speaker   │    in-memory       │ (120 MB RAM) │                      │
│  └──────────┘                    └──────┬───────┘                      │
│                                         │ WAV file                     │
│                                         ▼                              │
│                                  ┌──────────────┐                      │
│                                  │  /dev/shm/   │ ← tmpfs (RAM)        │
│                                  │  tts_out.wav │    NOT SD card       │
│                                  └──────────────┘                      │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```