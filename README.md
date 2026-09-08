# Jarvis — Your Private, Local AI Assistant

Everything below runs **on your own Windows PC**. Your voice, conversations,
and memory never leave your machine. The only exceptions are the optional
weather and web-search tools — and even those only send the specific query
text out, never your conversation history.

## What's Local vs What Touches the Internet

| Component        | Runs on your PC | Touches internet |
|-------------------|:---:|:---:|
| Speech-to-text (Whisper) | ✅ | ❌ |
| LLM "brain" (Ollama)     | ✅ | ❌ |
| Memory (SQLite file)     | ✅ | ❌ |
| Text-to-speech (default) | ✅ | ❌ |
| Weather tool              | — | ✅ (city name only) |
| Web search tool           | — | ✅ (search query only) |

You can disable the two internet tools entirely in `config.py` (`ENABLE_WEATHER`,
`ENABLE_WEB_SEARCH`) for a 100% offline assistant.

## Step 1 — Install Python

Install **Python 3.10 or 3.11** from python.org. During install, check
"Add Python to PATH."

## Step 2 — Install Ollama (the local LLM engine)

1. Download and install from https://ollama.com/download (Windows installer).
2. Open Command Prompt / PowerShell and pull a model:
   ```
   ollama pull llama3.1:8b
   ```
   This downloads once (~4.7GB) and then runs entirely offline forever after.
   With your RTX GPU this will run smoothly.
3. Leave Ollama running in the background (it auto-starts a local server at
   `http://localhost:11434`).

## Step 3 — Set Up the Project

1. Copy the `jarvis` folder anywhere on your PC, e.g. `C:\Users\YourName\jarvis`.
2. Open Command Prompt in that folder and create a virtual environment:
   ```
   python -m venv venv
   venv\Scripts\activate
   ```
3. Install dependencies:
   ```
   pip install -r requirements.txt
   ```
4. If you have an NVIDIA GPU (you do), also install CUDA-enabled PyTorch
   dependencies for faster-whisper acceleration — usually detected
   automatically, but if STT is slow, follow faster-whisper's GPU setup
   notes at: https://github.com/SYSTRAN/faster-whisper#gpu

## Step 4 — (Optional) Weather API Key

For the weather tool, get a free key at https://openweathermap.org/api,
then set it as an environment variable:
```
setx OPENWEATHER_API_KEY "your-key-here"
```
(Restart your terminal after this.) Skip this if you don't want weather —
just set `ENABLE_WEATHER = False` in `config.py`.

## Step 5 — Run It

**Text mode** (type instead of speak — good for testing):
```
python main.py text
```

**Voice mode** (speak and listen):
```
python main.py voice
```

## How It Works

1. You speak (or type) → converted to text locally by Whisper.
2. Text is sent to your local Ollama model along with recent conversation
   history and any tools it might need (weather, search, opening apps, etc.).
3. The model either answers directly or requests a tool call.
4. If a tool is needed, it runs locally (or makes a minimal, specific web
   request) and the result is fed back to the model for a final answer.
5. The reply is spoken back to you via local TTS.
6. Every exchange is saved to `data/memory.db` on your disk so it can be
   recalled in future sessions.

## Customizing

- **Change the model:** edit `OLLAMA_MODEL` in `config.py`. Try `mistral`,
  `qwen2.5:7b`, or a larger model like `llama3.1:70b` if your GPU has the VRAM.
- **Change personality:** edit `SYSTEM_PROMPT` in `config.py`.
- **Add new tools:** add a function in `modules/tools.py`, register it in
  `TOOL_DEFINITIONS` in `modules/llm.py`, and add a dispatch line in
  `execute_tool()`.
- **Nicer voice:** set `TTS_ENGINE = "edge"` in `config.py` and
  `pip install edge-tts playsound`. Note: this sends spoken text (not your
  data/history) to Microsoft's servers.
- **Wake word ("Hey Jarvis"):** for a hands-free always-listening mode,
  add `openwakeword` or `pvporcupine` — ask me and I'll wire it in.

## Advanced Setup: Smart Home, Phone Control & WhatsApp

These three integrations are now built in. Each is independent — set up
whichever you want, and leave the others disabled in `config.py` if you're
not ready for them yet.

### A) Smart Home (your plugs/bulbs) via Home Assistant

You don't have a hub yet, so we're using **Home Assistant** — free, runs
locally, and supports nearly every plug/bulb brand (Tuya/Smart Life, Kasa,
generic WiFi devices, etc.) through local integrations.

1. Install **Docker Desktop** for Windows: https://www.docker.com/products/docker-desktop
2. Open Command Prompt and run:
   ```
   docker run -d --name homeassistant --restart=unless-stopped --network=host -v C:\ha-config:/config homeassistant/home-assistant:stable
   ```
   (If `--network=host` doesn't work on Windows, use `-p 8123:8123` instead.)
3. Open `http://localhost:8123` in your browser and complete the setup wizard
   (create an account — this stays local to your PC).
4. Go to **Settings → Devices & Services → Add Integration**, search for your
   plug brand (e.g. "Tuya", "TP-Link Kasa", "Smart Life") and follow the
   prompts to link your existing devices. Most ask for the same login you
   use in the device's own app the first time, then control happens locally
   after that.
5. Generate an access token: click your profile (bottom left) → scroll to
   **Long-Lived Access Tokens** → **Create Token**. Copy it.
6. Set it as an environment variable:
   ```
   setx HA_TOKEN "paste-your-token-here"
   ```
   Restart your terminal after this.
7. In `config.py`, confirm `HA_URL = "http://localhost:8123"` (or your HA
   device's IP if running elsewhere on your network).

Try it: *"turn on the bedroom light"*, *"list my devices"*.

### B) Android Phone Control via Wireless ADB

This uses Android's own local debugging protocol over your WiFi — no app
to install, no account, no cloud.

1. Install **platform-tools**: https://developer.android.com/tools/releases/platform-tools
   — unzip it somewhere like `C:\platform-tools` and add that folder to your
   Windows PATH (Search "Edit environment variables" → Path → New).
2. On your phone: **Settings → About phone** → tap "Build number" 7 times to
   enable Developer Options.
3. Go to **Settings → System → Developer options** → enable **Wireless debugging**.
4. Tap **Wireless debugging → Pair device with pairing code**. It'll show an
   IP:port and a 6-digit code.
5. On your PC:
   ```
   adb pair <ip>:<port>
   ```
   enter the 6-digit code when prompted.
6. Then connect (note: this is usually a *different* port than pairing —
   shown on the main Wireless Debugging screen):
   ```
   adb connect <ip>:<port>
   ```
7. Copy that `ip:port` into `config.py` as `ADB_DEVICE_ADDRESS`.
8. Both phone and PC must stay on the **same WiFi network** for this to work.

Try it: *"what's my phone battery"*, *"open spotify on my phone"*, *"call [contact]"*.

⚠️ Note: `phone_send_sms` opens the message pre-filled on your phone but
doesn't auto-tap "send" — that's a deliberate safety choice so a
misheard voice command can never send a text without you confirming it.

### C) WhatsApp Messaging (on-demand)

This drives your own WhatsApp Web session locally via a browser Jarvis
controls — the same thing you'd do manually, just automated.

1. Install **Google Chrome** if you don't have it.
2. Download **ChromeDriver** matching your Chrome version:
   https://googlechromelabs.github.io/chrome-for-testing/ — place `chromedriver.exe`
   somewhere on your PATH (e.g. the same `C:\platform-tools` folder from step B).
3. First time you ask it to send a WhatsApp message, a Chrome window will
   open showing a QR code — scan it with your phone's WhatsApp
   (**WhatsApp → Settings → Linked Devices → Link a Device**) exactly like
   using WhatsApp Web normally.
4. After that first scan, your session is saved in `data/whatsapp_profile/`
   on your own disk, so it won't ask again (until WhatsApp logs you out,
   same as normal WhatsApp Web behavior).

Try it: *"send a whatsapp message to Mom saying I'll be home by 8"*.

### Try It All Together

```
python main.py voice
```
- "Turn off the living room plug"
- "Send a WhatsApp to Raj saying running 10 minutes late"
- "What's my phone battery at"
- "Open WhatsApp on my phone"

## Full Roadmap: Where This Sits & What's Next

You now have Stages 1–5 of the full roadmap: foundations, voice I/O, LLM
brain with memory, tool use, and real device/IoT integrations. Natural next
upgrades:

- **RAG over your own files** — let it answer questions about your personal
  documents/notes using a local vector database (Chroma) — fully private.
- **Wake word detection** — always-listening hands-free mode ("Hey Jarvis").
- **GUI/dashboard** — a simple visual interface instead of the terminal.
- **Auto-reply on WhatsApp** — if you later want it reading and replying to
  incoming messages automatically (you said on-demand only for now).
- **More phone actions** — reading notifications, controlling media playback,
  taking photos remotely, etc. — all addable via the same ADB pattern.

Just ask and I'll build any of these next.
