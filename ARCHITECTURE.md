# AI Content Generation - Architecture Documentation

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Provider System](#provider-system)
3. [Pipeline Orchestration](#pipeline-orchestration)
4. [Package Structure](#package-structure)
5. [CLI Commands Reference](#cli-commands-reference)
6. [Presets Reference](#presets-reference)

---

## Architecture Overview

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLI Layer                                       │
│                         (cli/main.py)                                        │
│    Commands: music, video, list-providers, list-presets, jobs, etc.         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Pipelines Layer                                    │
│                        (pipelines/*.py)                                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────────┐  │
│  │  MusicPipeline  │  │  VideoPipeline  │  │    FullContentPipeline      │  │
│  │                 │  │                 │  │                             │  │
│  │ • performance_  │  │ • text_to_video │  │ 1. Music + Image (parallel) │  │
│  │   first         │  │ • image_to_     │  │ 2. Video generation         │  │
│  │ • lyrics_first  │  │   video         │  │ 3. FFmpeg merge             │  │
│  │ • reference_    │  │ • style_preset  │  │ 4. Upload (YouTube/S3)      │  │
│  │   based         │  │                 │  │                             │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Provider Registry                                   │
│                       (core/registry.py)                                     │
│                                                                              │
│   Decorator-based registration: @ProviderRegistry.register_music("name")    │
│   Lazy singleton instantiation: get_music("name") → cached instance         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Provider Implementations                             │
│                          (providers/*/*.py)                                  │
│                                                                              │
│  MUSIC: GoogleLyria, MiniMaxMusic                                           │
│  VIDEO: GoogleVeo, KlingDirect, AIMLAPIKling                                │
│  IMAGE: GoogleImagen                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           External APIs                                      │
│                                                                              │
│   Google GenAI (Lyria, Veo, Imagen)                                         │
│   AIMLAPI (MiniMax Music, Kling proxy)                                      │
│   KlingAI Direct (JWT authentication)                                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Data Flow

```
User Input (CLI)
      │
      ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Presets   │────▶│  Pipeline   │────▶│  Provider   │
│  (optional) │     │ Orchestrator│     │  Registry   │
└─────────────┘     └─────────────┘     └─────────────┘
                                              │
                                              ▼
                                   ┌─────────────────────┐
                                   │  Provider Instance  │
                                   │  (Lyria/Veo/Kling)  │
                                   └─────────────────────┘
                                              │
                                              ▼
                                   ┌─────────────────────┐
                                   │    External API     │
                                   │  (Google/AIMLAPI)   │
                                   └─────────────────────┘
                                              │
                                              ▼
                                   ┌─────────────────────┐
                                   │  GenerationResult   │
                                   │  (unified output)   │
                                   └─────────────────────┘
                                              │
                                              ▼
                                   ┌─────────────────────┐
                                   │   Output File       │
                                   │  (exports/*.mp4)    │
                                   └─────────────────────┘
```

---

## Provider System

### Plugin Architecture

The provider system uses a **decorator-based plugin architecture** that enables:
- Self-registration at import time
- No manual configuration required
- Easy extensibility for new providers

```python
# Example: Registering a new music provider
@ProviderRegistry.register_music("my-provider")
class MyMusicProvider:
    name = "my-provider"
    supports_vocals = True
    supports_realtime = False
    supports_reference_audio = False
    
    async def generate(self, prompt, **kwargs) -> GenerationResult:
        # Implementation
        pass
```

### Provider Protocols

All providers implement a **Protocol** (structural subtyping) that defines the contract:

#### MusicProvider Protocol

| Property | Type | Description |
|----------|------|-------------|
| `name` | `str` | Unique identifier |
| `supports_vocals` | `bool` | Can generate vocals |
| `supports_realtime` | `bool` | Supports streaming |
| `supports_reference_audio` | `bool` | Supports style transfer |

| Method | Parameters | Returns |
|--------|------------|---------|
| `generate()` | `prompt`, `bpm`, `duration_seconds`, `lyrics`, `reference_audio_url`, `output_path` | `GenerationResult` |

#### VideoProvider Protocol

| Property | Type | Description |
|----------|------|-------------|
| `name` | `str` | Unique identifier |
| `supports_image_to_video` | `bool` | Can animate images |
| `max_duration_seconds` | `int` | Maximum video length |

| Method | Parameters | Returns |
|--------|------------|---------|
| `generate()` | `prompt`, `aspect_ratio`, `duration_seconds`, `first_frame_url`, `output_path` | `GenerationResult` |

### Available Providers

#### Music Providers

| Provider | Name | Features | Output |
|----------|------|----------|--------|
| **Google Lyria** | `lyria` | Real-time WebSocket streaming, BPM control, instrumental only | WAV |
| **MiniMax** | `minimax` | Async polling, lyrics support, reference audio, vocals | MP3 |

#### Video Providers

| Provider | Name | Features | Speed |
|----------|------|----------|-------|
| **Google Veo** | `veo` | Text/image-to-video, multiple aspect ratios | ~30 seconds |
| **Kling Direct** | `kling` | Highest quality, JWT auth, v2.1-master model | 5-14 minutes |
| **AIMLAPI Kling** | `aimlapi-kling` | Uses AIMLAPI credentials, rate limit handling | 5-14 minutes |

### Unified Result Contract

All providers return the same `GenerationResult` structure:

```python
@dataclass
class GenerationResult:
    success: bool              # Whether generation succeeded
    provider: str              # Provider name (e.g., "lyria")
    content_type: str          # "music", "video", or "image"
    file_path: Path | None     # Path to saved file
    data: bytes | None         # Raw binary data
    error: str | None          # Error message if failed
    generation_id: str | None  # For async polling
    duration_seconds: float    # Content duration
    file_size_mb: float        # File size
    metadata: dict             # Additional info
```

---

## Pipeline Orchestration

### Pipeline Components

```
PipelineConfig                  PipelineResult
┌─────────────────┐            ┌─────────────────────────────┐
│ output_dir      │            │ success: bool               │
│ parallel: bool  │            │ outputs: {key: GenResult}   │
│ stop_on_error   │            │ errors: [str]               │
│ cleanup_on_fail │            │ started_at / completed_at   │
└─────────────────┘            │ duration_seconds            │
                               │ metadata: {}                │
                               └─────────────────────────────┘
```

### Music Pipeline Strategies

| Strategy | Method | Provider | Use Case |
|----------|--------|----------|----------|
| **Performance-First** | `performance_first()` | Lyria | Generate instrumental → write lyrics later |
| **Lyrics-First** | `lyrics_first()` | MiniMax | Structure lyrics → generate with vocals |
| **Reference-Based** | `reference_based()` | MiniMax | Style transfer from reference audio |

### Full Content Pipeline Flow

The `FullContentPipeline` orchestrates complete music video generation:

```
┌─────────────────────────────────────────────────────────────┐
│                    FullContentPipeline                       │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     │
   Phase 1 (Parallel)                               │
┌─────────────┐       ┌─────────────┐              │
│  Generate   │       │  Generate   │              │
│   Music     │       │  Keyframe   │              │
│  (Lyria)    │       │  (Imagen)   │              │
└──────┬──────┘       └──────┬──────┘              │
       │                     │                     │
       │    asyncio.gather() │                     │
       └──────────┬──────────┘                     │
                  │                                │
                  ▼                                │
           Phase 2                                 │
    ┌─────────────────────┐                       │
    │   Generate Video    │◄──────────────────────┘
    │   (Veo/Kling)       │     (uses keyframe)
    │   from keyframe     │
    └──────────┬──────────┘
               │
               ▼
          Phase 3
    ┌─────────────────────┐
    │   FFmpeg Merge      │
    │   Audio + Video     │
    └──────────┬──────────┘
               │
               ▼
          Phase 4
    ┌─────────────────────┐
    │   Upload            │
    │   (YouTube/S3)      │
    └─────────────────────┘
```

### Key Orchestration Features

1. **Parallel Execution**: Music and keyframe image generated simultaneously via `asyncio.gather()`
2. **Graceful Degradation**: If image generation fails, falls back to text-to-video
3. **Error Aggregation**: All errors collected in `PipelineResult.errors[]`
4. **Timing Metrics**: Automatic duration tracking via `started_at`/`completed_at`
5. **Optional Phases**: FFmpeg merge and upload are conditional
6. **Config-Driven**: `PipelineConfig` controls parallel/sequential, error handling, cleanup

---

## Package Structure

```
src/ai_content/
├── cli/                    # Command-line interface
│   ├── __init__.py
│   └── main.py            # Typer CLI commands
│
├── config/                 # Configuration management
│   ├── __init__.py
│   ├── loader.py          # YAML config loading
│   └── settings.py        # Pydantic settings models
│
├── core/                   # Core abstractions
│   ├── __init__.py
│   ├── exceptions.py      # Custom exceptions
│   ├── job_tracker.py     # Async job tracking
│   ├── provider.py        # Provider protocols
│   ├── registry.py        # Provider registry
│   └── result.py          # GenerationResult
│
├── integrations/           # External service integrations
│   ├── __init__.py
│   ├── archive.py         # Internet Archive
│   ├── media.py           # FFmpeg processing
│   └── youtube.py         # YouTube upload
│
├── pipelines/              # Orchestrated workflows
│   ├── __init__.py
│   ├── base.py            # PipelineResult, PipelineConfig
│   ├── full.py            # FullContentPipeline
│   ├── music.py           # MusicPipeline
│   └── video.py           # VideoPipeline
│
├── presets/                # Pre-configured styles
│   ├── __init__.py
│   ├── music.py           # Music presets (jazz, blues, etc.)
│   └── video.py           # Video presets (nature, urban, etc.)
│
├── providers/              # AI provider implementations
│   ├── __init__.py
│   ├── aimlapi/           # AIMLAPI-based providers
│   │   ├── __init__.py
│   │   ├── client.py      # Base HTTP client
│   │   ├── minimax.py     # MiniMax music
│   │   └── video.py       # AIMLAPI Kling video
│   ├── google/            # Google AI providers
│   │   ├── __init__.py
│   │   ├── imagen.py      # Image generation
│   │   ├── lyria.py       # Music generation
│   │   └── veo.py         # Video generation
│   └── kling/             # Direct Kling API
│       ├── __init__.py
│       └── direct.py      # JWT-authenticated client
│
└── utils/                  # Utility functions
    ├── __init__.py
    ├── file_handlers.py   # File operations
    ├── lyrics_parser.py   # Lyrics structure parsing
    └── retry.py           # Retry logic
```

---

## CLI Commands Reference

### Global Options

| Option | Short | Description |
|--------|-------|-------------|
| `--verbose` | `-v` | Enable debug logging |
| `--config` | `-c` | Path to config file |

### Commands

#### `music` - Generate Music

```bash
uv run ai-content music [OPTIONS]
```

| Option | Short | Description | Default |
|--------|-------|-------------|---------|
| `--prompt` | `-p` | Music style prompt | Required (unless `--style`) |
| `--style` | `-s` | Preset style name | None |
| `--provider` | | Provider: lyria, minimax | lyria |
| `--duration` | `-d` | Duration in seconds | 30 |
| `--bpm` | | Beats per minute | 120 |
| `--lyrics` | `-l` | Path to lyrics file | None |
| `--reference-url` | `-r` | Reference audio URL | None |
| `--output` | `-o` | Output file path | Auto-generated |
| `--force` | `-f` | Force even if duplicate | False |

#### `video` - Generate Video

```bash
uv run ai-content video [OPTIONS]
```

| Option | Short | Description | Default |
|--------|-------|-------------|---------|
| `--prompt` | `-p` | Scene description | Required (unless `--style`) |
| `--style` | `-s` | Preset style name | None |
| `--provider` | | Provider: veo, kling, aimlapi-kling | veo |
| `--aspect` | `-a` | Aspect ratio | 16:9 |
| `--duration` | `-d` | Duration in seconds | 5 |
| `--image` | `-i` | First frame image | None |
| `--output` | `-o` | Output file path | Auto-generated |

#### Other Commands

| Command | Description |
|---------|-------------|
| `list-providers` | List all available providers |
| `list-presets` | List all available presets |
| `music-status <id>` | Check MiniMax generation status |
| `jobs` | List tracked generation jobs |
| `jobs-stats` | Show job statistics |
| `jobs-sync` | Sync pending jobs from API |

---

## Presets Reference

### Music Presets

| Preset | BPM | Mood | Style Description |
|--------|-----|------|-------------------|
| `jazz` | 95 | nostalgic | Smooth jazz fusion, walking bass, mellow saxophone |
| `blues` | 72 | soulful | Delta blues, bluesy guitar, vintage amplifier |
| `ethiopian-jazz` | 85 | mystical | Ethio-jazz fusion, Mulatu Astatke inspired |
| `cinematic` | 100 | epic | Orchestral, Hans Zimmer inspired |
| `electronic` | 128 | euphoric | Progressive house, festival anthem |
| `ambient` | 60 | peaceful | Brian Eno inspired, meditative |
| `lofi` | 85 | relaxed | Lo-fi hip-hop, study beats |
| `rnb` | 90 | sultry | Contemporary R&B, neo-soul |
| `salsa` | 180 | fiery | Cuban salsa dura, Fania Records inspired |
| `bachata` | 130 | romantic | Dominican bachata, Romeo Santos inspired |
| `kizomba` | 95 | sensual | Angolan kizomba, zouk influence |

### Video Presets

| Preset | Aspect Ratio | Style Description |
|--------|--------------|-------------------|
| `nature` | 16:9 | Documentary, wildlife, golden hour |
| `urban` | 21:9 | Cyberpunk, neon, Blade Runner inspired |
| `space` | 16:9 | Sci-fi, contemplative, Interstellar inspired |
| `abstract` | 1:1 | Commercial, satisfying, liquid metal |
| `ocean` | 16:9 | Underwater, paradise, vibrant colors |
| `fantasy` | 21:9 | Epic, dragon, Lord of the Rings inspired |
| `portrait` | 9:16 | Fashion, beauty, studio lighting |

---

## Environment Configuration

Required environment variables in `.env`:

```bash
# Google APIs (Lyria, Veo, Imagen)
GEMINI_API_KEY=your_google_api_key

# AIMLAPI (MiniMax, AIMLAPI-Kling)
AIMLAPI_KEY=your_aimlapi_key

# KlingAI Direct API (optional)
KLINGAI_API_KEY=your_kling_api_key
KLINGAI_SECRET_KEY=your_kling_secret_key

# Optional
LOG_LEVEL=INFO
```

---

## Error Handling

### Rate Limiting

All providers include exponential backoff retry logic for 429 errors:

```
Attempt 1: Wait 2s
Attempt 2: Wait 4s
Attempt 3: Wait 8s
Attempt 4: Wait 16s
Attempt 5: Wait 32s (max 60s)
```

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `429 Too Many Requests` | Rate limit or quota exhausted | Wait for reset or upgrade plan |
| `403 Forbidden` | Account verification required | Complete verification at provider |
| `401 Unauthorized` | Invalid API key | Check `.env` configuration |
| `RESOURCE_EXHAUSTED` | Google quota exceeded | Wait for daily reset |

---

*Generated: February 2026*
