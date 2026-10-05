# 360° Immersive Room with ESP32 Scent Diffusers

A voice-driven multisensory room. A visitor describes a place out loud, and within about a minute the room shows it as an 8K 360° panorama on four projectors, plays matching ambient sound and releases a matching scent.

Built as a Howest CTAI team project (team Ergo G1, 2025–2026) for the occupational-therapy track.

![A generated 360° panorama, shown flat: waterfall in a forest](docs/panorama.jpg)

Demo video and more photos: [xchocapic.github.io/projects/immersive-room.html](https://xchocapic.github.io/projects/immersive-room.html)

## How it works

1. **Voice to prompt.** A recording is sent to an Azure Function. Azure Speech transcribes it and GPT-4o-mini turns it into an image prompt and a scene category (beach, forest, garden or mountain).
2. **Prompt to panorama.** ComfyUI on a campus GPU server generates a seamless 360° image with SDXL (JuggernautXL v9 + a 360° LoRA) and upscales it 4× with ESRGAN to 8192 × 4096. The result goes to Azure Blob Storage.
3. **Sound.** A matching ambient track for the category is picked from Blob Storage.
4. **Scent.** The backend calls a small Python Azure Function, which sends a cloud-to-device message through Azure IoT Hub to the ESP32 diffuser for that scene.
5. **Display.** The Unity front end (not in this repo) shows the panorama and plays the sound.

```
Voice ──▶ .NET Azure Functions ──▶ ComfyUI (SDXL) ──▶ 8K panorama + sound
                 │
                 ▼
        espendpoint (Python) ──▶ Azure IoT Hub ──MQTT/TLS──▶ 4 × ESP32 diffusers
```

## Scent diffusers

![How the scent diffusers work, from the room app to the fans](docs/scent-diffusers.png)

![One of the diffusers: yellow 3D-printed housing with a fan on top](docs/diffuser.jpg)

Four identical units, one per scene: `esp0-beach`, `esp1-forest`, `esp2-garden`, `esp3-mountain`.

- **Firmware:** C on ESP-IDF 5 with FreeRTOS (`Code/esp32/esp-diffuser/main`).
  - Joins Wi-Fi, with a fallback network.
  - Syncs time over SNTP and signs its own IoT Hub SAS token (HMAC-SHA256).
  - Connects over MQTT with TLS on port 8883.
- **Commands:** JSON messages such as `{"command": "on", "duration": 2}` switch the fans on for the given number of minutes; `"off"` stops them.
- **Hardware:** ESP32, relay module switching two fans (GPIO 27 and 26), powered from a power bank.
- **Enclosure:** 3D-printed housing and scent-bottle bracket, in `hardware/`.

| Scene | Scent | Sound |
|-------|-------|-------|
| Beach | Ocean breeze | Waves, seagulls |
| Forest | Pine, earth | Birds, rustling leaves |
| Garden | Floral | Bees, gentle wind |
| Mountain | Fresh alpine | Wind, distant streams |

## Repository layout

```
Code/
├── azure-pipeline/
│   ├── dotnet-functions/   .NET 8 Azure Functions (speech, prompt, image, sound)
│   └── *.py, *.sh          test and helper scripts
└── esp32/esp-diffuser/
    ├── main/               ESP32 firmware (C, ESP-IDF)
    └── espendpoint/        Python Azure Function that controls the diffusers
hardware/                   STL files for the diffuser housing and bottle bracket
Documentation/              project documentation and presentation
docs/                       images used in this README
```

## Setup

You need an Azure subscription (Functions, IoT Hub, Blob Storage, Speech, OpenAI), a ComfyUI server with the models above, ESP-IDF 5, the .NET 8 SDK and Python 3.10+.

**Backend (.NET Functions)**
```bash
cd Code/azure-pipeline/dotnet-functions
cp local.settings.example.json local.settings.json   # fill in your Azure and ComfyUI values
func start
```

**Diffuser endpoint (Python Function)**
```bash
cd Code/esp32/esp-diffuser/espendpoint
cp local.settings.example.json local.settings.json   # set IOT_HUB_CONNECTION_STRING
pip install -r requirements.txt
func start
```

**ESP32 firmware**
```bash
cd Code/esp32/esp-diffuser
cp main/config.example.h main/config.h   # Wi-Fi, IoT Hub host, device ID and key
idf.py build flash monitor
```

The helper scripts in `Code/azure-pipeline` read the function key from the `AZURE_FUNCTION_KEY` environment variable. No credentials are stored in the repository.

## API

| Method | Route | Purpose |
|--------|-------|---------|
| POST | `/api/audio-to-image-with-sound` | WAV audio in, panorama URL, sound URL and category out |
| POST | `/api/audio-to-prompt` | WAV audio in, image prompt out |
| POST | `/api/generate-image` | prompt in, panorama URL out |
| GET | `/api/images/{category}` | random pre-made panorama |
| GET | `/api/sounds/{category}` | random ambient sound |
| GET | `/api/{device}/{minutes}` | diffuser endpoint: turn a diffuser on |
| GET | `/api/{device}/stop` · `/api/stop-all` | diffuser endpoint: turn one or all off |

## Team

Team Ergo G1, Howest University of Applied Sciences.

Mihai Alexandru Matei was product owner and built the backend, the ESP32 diffusers (electronics, firmware and enclosures) and the documentation. The Unity front end and design were built by the other team members.

## Tech stack

.NET 8 · Azure Functions · Azure OpenAI (GPT-4o-mini) · Azure Speech · Azure IoT Hub · Azure Blob Storage · ComfyUI · SDXL · Python · C / ESP-IDF · MQTT · Unity 6
