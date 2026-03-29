# vllm-stack

Local LLM Stack สำหรับรัน AI บนเครื่องตัวเอง โดยใช้ **Open WebUI** เป็น chat interface และเลือก inference backend ได้ 2 แบบ:

| Backend | เหมาะกับ | ข้อดี |
| ------- | -------- | ----- |
| **Ollama** ⭐ | Mac Apple Silicon | native ARM64, โหลดเร็ว, ใช้ RAM น้อย |
| **vLLM** | NVIDIA GPU (Linux/Windows) | throughput สูง, รองรับ concurrent requests |

## ภาพรวม

```text
┌─────────────────┐        ┌──────────────────────┐
│   Open WebUI    │──────▶ │  Ollama หรือ vLLM    │
│  (port 3500)    │  HTTP  │    (port 8000)        │
│  Chat Interface │        │  OpenAI-compatible    │
│                 │        │  Inference Engine     │
└─────────────────┘        └──────────────────────┘
                                       │
                                       ▼
                            ┌──────────────────────┐
                            │   LLM Model          │
                            │  (./ollama/ หรือ     │
                            │   ./models/ on disk) │
                            └──────────────────────┘
```

## การติดตั้ง (Setup)

### 1. Clone โปรเจค

```bash
git clone https://github.com/lumduan/vllm-stack.git
cd vllm-stack
```

### 2. ตั้งค่า Environment Variables

```bash
cp .env.sample .env
```

แก้ไขไฟล์ `.env`:

```env
HF_TOKEN=hf_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
MODEL_NAME=typhoon-ai/typhoon2.5-qwen3-4b
LLM_BACKEND=ollama    # ollama (แนะนำ Mac) หรือ vllm (NVIDIA GPU)
```

> สร้าง HuggingFace token ได้ที่ [huggingface.co/settings/tokens](https://huggingface.co/settings/tokens)

### 3. ตั้งค่า SSH Keys สำหรับ Ollama (ถ้าจำเป็น)

ถ้า Ollama ต้องการ SSH key ให้สร้างจาก sample:

```bash
cp ollama/id_ed25519.sample ollama/id_ed25519
cp ollama/id_ed25519.pub.sample ollama/id_ed25519.pub
```

แล้วแทนที่ content ด้วย key จริงที่ต้องการใช้ ไฟล์เหล่านี้จะไม่ถูก commit ขึ้น git

### 4. เลือก Backend ใน docker-compose.yaml

ไฟล์ `docker-compose.yaml` มี 2 service block ให้เลือก uncomment อันที่ต้องการ:

#### Ollama — แนะนำสำหรับ Mac Apple Silicon (default)

```yaml
# ✅ uncomment ส่วนนี้
ollama:
  image: ollama/ollama:latest
  platform: linux/arm64   # native ARM64
  ...

# ❌ comment ส่วน vllm ออก
# vllm:
#   ...
```

Ollama จะ pull โมเดลและสร้าง alias อัตโนมัติตอน start — ไม่ต้อง download ด้วยตนเอง

#### vLLM — สำหรับ NVIDIA GPU

```yaml
# ✅ uncomment ส่วนนี้
vllm:
  image: vllm/vllm-openai:latest   # GPU image
  runtime: nvidia
  ipc: host
  ...

# ❌ comment ส่วน ollama ออก
# ollama:
#   ...
```

> ต้องติดตั้ง [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html) ก่อน

### 4. รัน Stack

```bash
docker compose up -d
```

เวลาที่ใช้ครั้งแรก (รวม download โมเดล):

- **Ollama (Mac M-series):** ~2–5 นาที
- **vLLM CPU:** ~15–30 นาที
- **vLLM GPU:** ~5–10 นาที

### 5. เปิดใช้งาน

เมื่อ services พร้อมแล้ว เข้าใช้งาน Chat ได้ที่:

**<http://localhost:3500>**

## Services

| Service | URL | คำอธิบาย |
| ------- | --- | --------- |
| Open WebUI | <http://localhost:3500> | Chat interface |
| LLM API | <http://localhost:8000> | OpenAI-compatible REST API |

## การใช้งาน API

ทั้ง Ollama และ vLLM รองรับ OpenAI API format:

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="sk-local",  # ค่า placeholder ใดก็ได้
)

response = client.chat.completions.create(
    model="typhoon-ai/typhoon2.5-qwen3-4b",
    messages=[{"role": "user", "content": "สวัสดี"}],
)
print(response.choices[0].message.content)
```

## โครงสร้างไฟล์ (File Structure)

```text
vllm-stack/
├── docker-compose.yaml         # Config หลัก (Ollama + vLLM ในไฟล์เดียว)
├── .env                        # Environment variables (ไม่ขึ้น git)
├── .env.sample                 # ตัวอย่าง environment variables
├── ollama/
│   ├── id_ed25519              # SSH private key สำหรับ Ollama (ไม่ขึ้น git)
│   ├── id_ed25519.pub          # SSH public key สำหรับ Ollama (ไม่ขึ้น git)
│   ├── id_ed25519.sample       # ตัวอย่าง SSH private key
│   ├── id_ed25519.pub.sample   # ตัวอย่าง SSH public key
│   └── models/                 # โมเดล Ollama cache (ไม่ขึ้น git)
├── models/                     # โมเดล HuggingFace cache สำหรับ vLLM (ไม่ขึ้น git)
├── openwebui/                  # ข้อมูล Open WebUI (chat history, settings)
└── README.md
```

## คำสั่งที่ใช้บ่อย

```bash
# รัน stack
docker compose up -d

# ดู logs แบบ real-time
docker compose logs -f

# ดู logs เฉพาะ backend
docker compose logs -f ollama   # หรือ vllm

# หยุด stack
docker compose down

# ตรวจสอบสถานะ services
docker compose ps
```

## การแก้ไขปัญหา (Troubleshooting)

### Open WebUI เชื่อมต่อไม่ได้

- รอให้ backend healthcheck ผ่านก่อน ตรวจสอบด้วย `docker compose ps`
- ตรวจสอบว่า `LLM_BACKEND` ใน `.env` ตรงกับ service ที่ uncomment อยู่

### Mac: vLLM crash ทันที (`Failed to infer device type`)

- บน Mac ให้ใช้ Ollama แทน vLLM — vLLM ไม่รองรับ Apple Silicon GPU
- ถ้าจำเป็นต้องใช้ vLLM: ตรวจสอบว่าใช้ image `vllm/vllm-openai-cpu:latest` และเปิด Rosetta 2 ใน Docker Desktop

### Mac: Docker ไม่รัน image

- เปิดใช้งาน Rosetta 2 ใน Docker Desktop → Settings → General → "Use Rosetta for x86_64/amd64 emulation"
- ตรวจสอบว่า Docker Desktop version ≥ 4.25

### NVIDIA GPU ไม่ถูกตรวจพบ

- ตรวจสอบว่าติดตั้ง NVIDIA Container Toolkit แล้ว: `docker run --rm --gpus all nvidia/cuda:12.0-base nvidia-smi`
- ตรวจสอบว่า uncomment `runtime: nvidia` และ `ipc: host` ใน `docker-compose.yaml` แล้ว

### HuggingFace Token ไม่ถูกต้อง

- ตรวจสอบค่า `HF_TOKEN` ใน `.env` และสิทธิ์การเข้าถึงโมเดลที่ [huggingface.co](https://huggingface.co)

### vLLM ค้างที่ "Asynchronous scheduling is enabled" ไม่มีอะไรเกิดขึ้นต่อ

สาเหตุ: บางโมเดลบน HuggingFace เก็บ model weights ด้วย XET storage format ซึ่ง XET client ใน Docker container ดาวน์โหลด weights ไม่ได้ ทำให้ vLLM ค้างรอไฟล์ที่ไม่มีอยู่บน disk

> **แนะนำ:** ถ้าใช้ Mac ให้เปลี่ยนไปใช้ Ollama แทน — เร็วกว่าและไม่มีปัญหานี้

ตรวจสอบว่า snapshot มีไฟล์ `.safetensors` หรือไม่:

```bash
ls models/hub/models--*/snapshots/*/*.safetensors 2>/dev/null || echo "ไม่มี weights — ต้อง download ด้วยตนเอง"
```

แก้ไขด้วยการ download weights ผ่าน Python โดยตรงใน container:

```bash
# 1. หยุด vLLM ก่อน
docker compose down vllm

# 2. ลบ cache เก่าที่ไม่สมบูรณ์
rm -rf models/hub models/xet

# 3. รัน container ชั่วคราวเพื่อ download โมเดล
docker run --rm \
  -v "$(pwd)/models:/root/.cache/huggingface" \
  -e HF_TOKEN=${HF_TOKEN} \
  vllm/vllm-openai-cpu:latest \
  python3 -c "
from huggingface_hub import snapshot_download
import os
snapshot_download(
    'typhoon-ai/typhoon2.5-qwen3-4b',
    token=os.environ.get('HF_TOKEN'),
    ignore_patterns=['*.msgpack', '*.h5', 'flax_model*', 'tf_model*'],
)
print('Download complete')
"

# 4. รัน stack ตามปกติ
docker compose up -d
```

> โมเดล 4B มีขนาดประมาณ **8GB** ใช้เวลา download ขึ้นกับความเร็ว internet

## License

MIT
