import json
import random
import time
import gevent
import locust
from locust import User, task, events, constant_throughput
import websocket

# Averages up to 10 seconds of audio when played, depends on the voice speed
DEFAULT_TEXT = (
    "Hello, this is a test message. I am testing if a long input will cause issues for the model "
    "like this sentence. "
)

TEXT_ARRAY = [
    "Hello.",
    "Hello, this is a test message.",
    DEFAULT_TEXT,
    DEFAULT_TEXT * 2,
    DEFAULT_TEXT * 3
]

# Custom command line arguments
@events.init_command_line_parser.add_listener
def on_parser_init(parser):
    parser.add_argument("--api-key", default="YOUR_API_KEY", help="API key for authentication")
    parser.add_argument("--encoding", default="mp3_22050_32", help="Encoding")
    parser.add_argument("--text", default=DEFAULT_TEXT, help="Text to use")
    parser.add_argument("--use-text-array", default="false", help="Text to use")
    parser.add_argument("--voice-id", default="aria", help="Text to use")


class WebSocketTTSUser(User):
    # Each user will send a request every 20 seconds, regardless of how long each request takes
    wait_time = constant_throughput(0.05)

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.api_key = self.environment.parsed_options.api_key
        self.voice_id = self.environment.parsed_options.voice_id
        self.text = self.environment.parsed_options.text
        self.encoding = self.environment.parsed_options.encoding
        self.use_text_array = self.environment.parsed_options.use_text_array
        if self.use_text_array:
            self.text = random.choice(TEXT_ARRAY)
        self.all_recieved = False

    @task
    def tts_task(self):
        # Do jitter waiting of up to 1 second
        # Users appear to be spawned every second so this ensures requests are not aligned
        gevent.sleep(random.random())

        max_wait_time = 10

        # Connection details
        uri = f"{self.environment.host}/v1/text-to-speech/{self.voice_id}/stream-input?auto_mode=true&output_format={self.encoding}"
        headers = {"xi-api-key": self.api_key}

        ws = None
        self.all_recieved = False
        try:
            init_msg = {"text": " "}
            # Use proper header format for websocket - this is case sensitive!
            ws = websocket.create_connection(uri, header=headers)
            ws.send(json.dumps(init_msg))

            # Start measuring after websocket initiated but before any messages are sent
            send_request_time = time.perf_counter()
            ws.send(json.dumps({"text": self.text}))

            # Send to flush and receive the audio
            ws.send(json.dumps({"text": ""}))

            def _receive():
                t_first_response = None
                audio_size = 0
                try:
                    while True:
                        # Wait up to 10 seconds for a response
                        ws.settimeout(max_wait_time)
                        response = ws.recv()
                        response_data = json.loads(response)

                        if "audio" in response_data and response_data["audio"]:
                            audio_size = audio_size + len(response_data["audio"])

                        if t_first_response is None:
                            t_first_response = time.perf_counter()
                            first_byte_ms = (
                                t_first_response - send_request_time
                            ) * 1000
                            if audio_size is None:
                                # The first response should always have audio
                                locust.events.request.fire(
                                    request_type="websocket",
                                    name="Bad Response (no audio)",
                                    response_time=first_byte_ms,
                                    response_length=audio_size,
                                    exception=Exception("Response has no audio"),
                                )
                                break

                        if "isFinal" in response_data and response_data["isFinal"]:
                            # Fire this event once finished streaming, but report the important TTFB metric
                            locust.events.request.fire(
                                request_type="websocket",
                                name="TTS Stream Success (First Byte)",
                                response_time=first_byte_ms,
                                response_length=audio_size,
                                exception=None,
                            )
                            break

                except websocket.WebSocketTimeoutException:
                    locust.events.request.fire(
                        request_type="websocket",
                        name="TTS Stream Timeout",
                        response_time=max_wait_time * 1000,
                        response_length=audio_size,
                        exception=Exception("Timeout waiting for response"),
                    )
                except Exception as e:
                    # Typically JSON decode error if the server returns HTTP backoff error
                    locust.events.request.fire(
                        request_type="websocket",
                        name="TTS Stream Failure",
                        response_time=0,
                        response_length=0,
                        exception=e,
                    )
                finally:
                    self.all_recieved = True

            gevent.spawn(_receive)

            # Sleep until recieved so new tasks aren't spawned
            while not self.all_recieved:
                gevent.sleep(1)

        except websocket.WebSocketTimeoutException:
            locust.events.request.fire(
                request_type="websocket",
                name="TTS Stream Timeout",
                response_time=max_wait_time * 1000,
                response_length=0,
                exception=Exception("Timeout waiting for response"),
            )
        except Exception as e:
            locust.events.request.fire(
                request_type="websocket",
                name="TTS Stream Failure",
                response_time=0,
                response_length=0,
                exception=e,
            )
        finally:
            # Try and close the websocket gracefully
            try:
                if ws:
                    ws.close()
            except Exception:
                pass



![LOGO](/logo.png)

# ElevenLabs Skills

Agent skills for [ElevenLabs](https://elevenlabs.io) developer products. These skills follow the [Agent Skills specification](https://agentskills.io/specification) and can be used with any compatible AI coding assistant.

## Installation

```bash
npx skills add elevenlabs/skills
```

## Available Skills

| Skill | Description |
|-------|-------------|
| [text-to-speech](./text-to-speech) | Convert text to lifelike speech using ElevenLabs' AI voices |
| [speech-to-text](./speech-to-text) | Transcribe audio files to text with timestamps |
| [speech-engine](./speech-engine) | Add real-time voice conversations to a custom LLM or chat agent |
| [agents](./agents) | Build conversational voice AI agents |
| [sound-effects](./sound-effects) | Generate sound effects from text descriptions |
| [music](./music) | Generate music tracks using AI composition |
| [voice-changer](./voice-changer) | Transform the voice in an audio recording into a different target voice (speech-to-speech) |
| [voice-isolator](./voice-isolator) | Remove background noise and isolate vocals/speech from audio |
| [dubbing](./dubbing) | Dub audio/video into other languages while preserving the original speakers' voices |
| [setup-api-key](./setup-api-key) | Guide through obtaining and configuring an ElevenLabs API key |

## Configuration

All skills require an ElevenLabs API key. Set it as an environment variable:

```bash
export ELEVENLABS_API_KEY="your-api-key"
```

Get your API key from the `setup-api-key` skill or use the [ElevenLabs dashboard](https://elevenlabs.io/app/settings/api-keys).

## SDK Support

Most skills include examples for:

- **Python** - `pip install elevenlabs`
- **JavaScript/TypeScript** - `npm install @elevenlabs/elevenlabs-js`
- **CLI** - `npm install -g @elevenlabs/cli` or `brew install elevenlabs/tap/elevenlabs` (wraps the REST API; reads `ELEVENLABS_API_KEY` automatically)

> **JavaScript SDK Warning:** Always use `@elevenlabs/elevenlabs-js`. Do not use `npm install elevenlabs` (that's an outdated v1.x package).

See the installation guide in any skill's `references/` folder for complete setup instructions including migration from deprecated packages.

## Evaluations

The `evals/` directory contains trigger and functional evaluations for all skills.

```bash
# Run all evaluations (trigger + functional)
python3 evals/run_all.py -v

# Trigger evals only — tests if skills fire for the right queries (~3 min)
python3 evals/run_all.py --trigger-only -v

# Functional evals only — tests if skills produce correct output (~15 min)
python3 evals/run_all.py --functional-only -v

# Specific skills
python3 evals/run_all.py --skills text-to-speech agents -v

# Custom model (see `cursor-agent --list-models`)
python3 evals/run_all.py --model gpt-5.4-high -v
```

Results are saved to `evals/results/<timestamp>/` with a `report.md` summary and `results.json` for programmatic access.

Functional evals use an isolated `cursor-agent` workspace per test case (under that results tree); they do **not** modify skill sources under each skill’s directory.

Requires the [Cursor Agent CLI](https://cursor.com/docs/cli/using) (`cursor-agent` on your `PATH`; override binary with `CURSOR_AGENT`) and Cursor authentication (`cursor-agent login` or `CURSOR_API_KEY`).

## License

MIT
https://elevenlabs.io/docs/overview/models#intended-purpose
