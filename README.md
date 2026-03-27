# vllm-stack

Local LLM Stack สำหรับรัน AI บนเครื่องตัวเอง โดยใช้ **vLLM** เป็น inference engine และ **Open WebUI** เป็น chat interface

รองรับทั้ง **CPU** (Mac Apple Silicon / ไม่มี GPU) และ **NVIDIA GPU** (Linux/Windows) — เลือกได้ใน `docker-compose.yaml` ไฟล์เดียว

## ภาพรวม

```
┌─────────────────┐        ┌──────────────────────┐
│   Open WebUI    │──────▶ │       vLLM           │
│  (port 3500)    │  HTTP  │    (port 8000)        │
│  Chat Interface │        │  OpenAI-compatible    │
│                 │        │  Inference Engine     │
└─────────────────┘        └──────────────────────┘
                                       │
                                       ▼
                            ┌──────────────────────┐
                            │   HuggingFace Model  │
                            │  (./models/ on disk) │
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
```

> สร้าง HuggingFace token ได้ที่ [huggingface.co/settings/tokens](https://huggingface.co/settings/tokens)

### 3. เลือก Mode ใน docker-compose.yaml

ไฟล์ `docker-compose.yaml` รองรับทั้ง CPU และ GPU ในไฟล์เดียว เลือก mode ด้วยการ **comment/uncomment** 3 จุด:

#### CPU Mode (default — Mac / ไม่มี GPU)

ค่า default ในไฟล์คือ CPU mode อยู่แล้ว ไม่ต้องแก้อะไร:

```yaml
# จุดที่ 1: image
image: vllm/vllm-openai-cpu:latest      # ✅ เปิด
# image: vllm/vllm-openai:latest        # ❌ ปิด

# จุดที่ 2: command
command:                                  # ✅ เปิด
  - --model
  - ...
  - --dtype
  - float32
  - --max-model-len
  - "2048"

# command:                               # ❌ ปิด (GPU section)
#   - --model
#   ...

# จุดที่ 3: GPU extras
# runtime: nvidia                        # ❌ ปิด
# ipc: host                             # ❌ ปิด
```

#### NVIDIA GPU Mode

แก้ไข `docker-compose.yaml` สลับ comment ที่ 3 จุด:

```yaml
# จุดที่ 1: image
# image: vllm/vllm-openai-cpu:latest    # ❌ ปิด
image: vllm/vllm-openai:latest          # ✅ เปิด

# จุดที่ 2: command — ปิด CPU section, เปิด GPU section
# command:                              # ❌ ปิด (CPU section)
#   ...

command:                                 # ✅ เปิด
  - --model
  - ...
  - --dtype
  - auto
  - --max-model-len
  - "4096"
  - --gpu-memory-utilization
  - "0.70"

# จุดที่ 3: GPU extras
runtime: nvidia                          # ✅ เปิด
ipc: host                               # ✅ เปิด
```

> **GPU เพิ่มเติม:** ต้องติดตั้ง [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html) ก่อน

### 4. รัน Stack

```bash
docker compose up -d
```

Docker จะดาวน์โหลดโมเดลอัตโนมัติในครั้งแรก โดยใช้เวลา:

- **CPU:** โหลดนาน 10–15 นาที (รอ healthcheck ผ่านก่อน)
- **GPU:** โหลดเร็วกว่า ประมาณ 5–7 นาที

### 5. เปิดใช้งาน

เมื่อ services พร้อมแล้ว เข้าใช้งาน Chat ได้ที่:

**<http://localhost:3500>**

## Services

| Service | URL | คำอธิบาย |
| ------- | --- | --------- |
| Open WebUI | <http://localhost:3500> | Chat interface |
| vLLM API | <http://localhost:8000> | OpenAI-compatible REST API |
| API Docs | <http://localhost:8000/docs> | Swagger UI สำหรับ vLLM API |

## การใช้งาน vLLM API

vLLM รองรับ OpenAI API format สามารถใช้กับ library ใดก็ได้ที่รองรับ OpenAI:

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
├── docker-compose.yaml         # Config หลัก (CPU + GPU ในไฟล์เดียว)
├── .env                        # Environment variables (ไม่ขึ้น git)
├── .env.sample                 # ตัวอย่าง environment variables
├── models/                     # โมเดล HuggingFace (ดาวน์โหลดอัตโนมัติ)
├── openwebui/                  # ข้อมูล Open WebUI (chat history, settings)
└── README.md
```

> ไดเรกทอรี `models/` และ `openwebui/` จะถูกสร้างอัตโนมัติเมื่อรัน Docker ครั้งแรก

## คำสั่งที่ใช้บ่อย

```bash
# รัน stack
docker compose up -d

# ดู logs แบบ real-time
docker compose logs -f

# ดู logs เฉพาะ vLLM
docker compose logs -f vllm

# หยุด stack
docker compose down

# ตรวจสอบสถานะ services
docker compose ps
```

## การแก้ไขปัญหา (Troubleshooting)

**vLLM ไม่เริ่มต้น / ใช้เวลานาน**

- โมเดลกำลังดาวน์โหลดหรือโหลดในครั้งแรก ดู logs ด้วย `docker compose logs -f vllm`
- CPU mode อาจใช้เวลาโหลดนานถึง 10–15 นาที อย่าเพิ่งปิด

**Mac: vLLM crash ทันที (`Failed to infer device type`)**

- ตรวจสอบว่า `docker-compose.yaml` ใช้ image `vllm/vllm-openai-cpu:latest` (ไม่ใช่ `vllm/vllm-openai:latest`)
- `vllm/vllm-openai:latest` เป็น CUDA image — crash บน Mac เพราะไม่มี CUDA

**Mac: Docker ไม่รัน image vLLM**

- เปิดใช้งาน Rosetta 2 ใน Docker Desktop → Settings → General → "Use Rosetta for x86_64/amd64 emulation"
- ตรวจสอบว่า Docker Desktop version ≥ 4.25

**NVIDIA GPU ไม่ถูกตรวจพบ**

- ตรวจสอบว่าติดตั้ง NVIDIA Container Toolkit แล้ว: `docker run --rm --gpus all nvidia/cuda:12.0-base nvidia-smi`
- ตรวจสอบว่า uncomment `runtime: nvidia` และ `ipc: host` ใน `docker-compose.yaml` แล้ว

**Open WebUI เชื่อมต่อไม่ได้**

- รอให้ vLLM healthcheck ผ่านก่อน แล้วตรวจสอบด้วย `docker compose ps`

**HuggingFace Token ไม่ถูกต้อง**

- ตรวจสอบค่า `HF_TOKEN` ใน `.env` และสิทธิ์การเข้าถึงโมเดลที่ [huggingface.co](https://huggingface.co)

## License

MIT
