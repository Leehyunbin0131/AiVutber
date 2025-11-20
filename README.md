# Discord Real-time STT Bot (High Performance)

This project is a high-performance Discord bot that performs real-time Speech-to-Text (STT) on user audio in voice channels. It uses **Faster-Whisper** for transcription and **Silero VAD** for voice activity detection, running in a separate process to ensure low latency and stability.

## Key Features

-   **Real-time Transcription**: Uses `faster-whisper` (large-v3-turbo) for high accuracy and speed.
-   **Low Latency**: Optimized pipeline with Ring Buffers and aggressive VAD settings (~0.2s latency).
-   **Multi-User Support**: Handles multiple users speaking simultaneously without mixing audio.
-   **Robust Architecture**:
    -   **Multiprocessing**: STT engine runs in a separate process to prevent bot freezing.
    -   **Auto Cleanup**: Automatically clears memory for inactive users to prevent leaks.
    -   **Token Security**: Uses `.env` for secure token management.
-   **Korean Optimized**: Configured specifically for Korean language transcription.

## Installation

### Prerequisites
-   Python 3.10+
-   NVIDIA GPU (Recommended) with CUDA installed.
-   [FFmpeg](https://ffmpeg.org/download.html) (Required for audio processing).

### Setup
1.  **Clone the repository**:
    ```bash
    git clone <repository_url>
    cd <repository_name>
    ```

2.  **Install Dependencies**:
    ```bash
    pip install -r requirements.txt
    ```
    *Note: This will install `torch`, `torchaudio`, and `faster-whisper` which are large packages.*

3.  **Configure Environment**:
    -   Create a `.env` file in the root directory.
    -   Add your Discord Bot Token:
        ```env
        DISCORD_TOKEN=your_discord_bot_token_here
        ```

4.  **Run the Bot**:
    ```bash
    python bot.py
    ```

## Usage Guide

1.  **Invite the Bot**: Invite the bot to your server using the OAuth2 URL generated from the Discord Developer Portal.
2.  **Join Voice Channel**: Join a voice channel.
3.  **Summon Bot**: Type `!join` in a text channel. The bot will join your voice channel and start listening.
4.  **Speak**: Speak naturally. The bot will print the transcription to the console (JSON format).
5.  **Leave**: Type `!leave` to disconnect the bot.

## Configuration (`config.py`)

All adjustable settings are located in `config.py`. You can modify these values to customize the bot's behavior.

### STT Settings
-   `STT_MODEL_ID`: The Whisper model to use (e.g., `deepdml/faster-whisper-large-v3-turbo-ct2`, `base`, `small`).
-   `STT_DEVICE`: `cuda` (GPU) or `cpu`.
-   `STT_COMPUTE_TYPE`: `float16` (GPU) or `int8` (CPU).
-   `STT_BEAM_SIZE`: Beam size for decoding. Lower (1) is faster, higher (5) is more accurate.

### VAD Settings (Voice Activity Detection)
-   `FRAME_DURATION_MS`: Frame size for VAD (Default: 32ms for Silero).
-   `RING_BUFFER_SIZE`: Number of frames to keep as context before speech (Pre-recording buffer). Increase this if the first syllable is being cut off.

### Cleanup
-   `USER_TIMEOUT_SECONDS`: Time in seconds before an inactive user's buffer is cleared from memory.

## Technical Architecture

### Pipeline
1.  **Audio Receiving (`bot.py`)**:
    -   Receives 48kHz Stereo PCM audio from Discord.
    -   Converts to 16kHz Mono PCM (required by models).
    -   Pushes audio chunks to a `multiprocessing.Queue`.

2.  **STT Processing (`stt_handler.py`)**:
    -   Runs in a separate **Process** to avoid blocking the Discord gateway heartbeat.
    -   **Ring Buffer**: Stores recent audio frames to provide context.
    -   **Silero VAD**: Detects voice activity with high accuracy.
    -   **Accumulation**: Buffers audio while the user is speaking.
    -   **Transcription**: When silence is detected, the accumulated audio is sent to `Faster-Whisper`.

3.  **Output**:
    -   Transcription results are sent back to the main process via a `result_queue`.
    -   The main process prints the result as JSON.

### Directory Structure
```
.
├── bot.py              # Main Discord bot entry point
├── stt_handler.py      # STT processing logic (Separate Process)
├── config.py           # Configuration file
├── requirements.txt    # Python dependencies
├── .env                # Environment variables (Token)
└── README.md           # Documentation
```

## Customization

### Changing the Model
To use a different model (e.g., English only or a smaller model), edit `config.py`:
```python
STT_MODEL_ID = "base.en" # For English base model
STT_LANGUAGE = "en"
```

### Adjusting Sensitivity
If the bot cuts off too early or picks up too much noise, adjust `config.py`:
-   **VAD Threshold**: Silero VAD is pre-tuned, but you can adjust `RING_BUFFER_SIZE` to capture more context.
-   **Silence Duration**: Currently hardcoded logic in `stt_handler.py` determines end-of-speech. You can modify the logic in `run_stt_process` to wait for more silence frames.
