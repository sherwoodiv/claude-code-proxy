# Anthropic API Proxy for Gemini, OpenAI, Ollama & Custom Models 🔄

**Use Anthropic clients (like Claude Code) with Gemini, OpenAI, Ollama, corporate APIs, or direct Anthropic backends.** 🤝

A proxy server that lets you use Anthropic clients with Gemini, OpenAI, local Ollama models, custom/corporate APIs, or Anthropic models themselves (a transparent proxy of sorts), all via LiteLLM. 🌉


![Anthropic API Proxy](pic.png)

## Quick Start ⚡

### Prerequisites

- [uv](https://github.com/astral-sh/uv) installed
- **One of the following** (depending on your preferred provider):
  - OpenAI API key 🔑
  - Google AI Studio (Gemini) API key 🔑
  - Google Cloud Project with Vertex AI API enabled (for ADC) ☁️
  - Local Ollama installation 🦙
  - Corporate/custom OpenAI-compatible API endpoint 🏢

### Setup 🛠️

#### From source

1. **Clone this repository**:
   ```bash
   git clone https://github.com/1rgs/claude-code-proxy.git
   cd claude-code-proxy
   ```

2. **Install uv** (if you haven't already):
   ```bash
   curl -LsSf https://astral.sh/uv/install.sh | sh
   ```
   *(`uv` will handle dependencies based on `pyproject.toml` when you run the server)*

3. **Configure Environment Variables**:
   Copy the example environment file:
   ```bash
   cp .env.example .env
   ```
   Edit `.env` and fill in your API keys and model configurations:

   *   `ANTHROPIC_API_KEY`: (Optional) Needed only if proxying *to* Anthropic models.
   *   `OPENAI_API_KEY`: Your OpenAI API key (Required if using the default OpenAI preference).
   *   `GEMINI_API_KEY`: Your Google AI Studio (Gemini) API key (Required if `PREFERRED_PROVIDER=google` and not using Vertex AI).
   *   `USE_VERTEX_AUTH` (Optional): Set to `true` to use Application Default Credentials (ADC) for Gemini. Note: when `USE_VERTEX_AUTH=true`, you must configure `VERTEX_PROJECT` and `VERTEX_LOCATION`.
   *   `VERTEX_PROJECT` (Optional): Your Google Cloud Project ID (Required if using Vertex AI).
   *   `VERTEX_LOCATION` (Optional): The Google Cloud region for Vertex AI (e.g., `us-central1`).
   *   `PREFERRED_PROVIDER` (Optional): Set to `openai` (default), `google`, `anthropic`, `ollama`, or `custom`. This determines the primary backend for mapping `haiku`/`sonnet`.
   *   `BIG_MODEL` (Optional): The model to map `sonnet` requests to. Defaults to `gpt-4.1`. Ignored when `PREFERRED_PROVIDER=anthropic`.
   *   `SMALL_MODEL` (Optional): The model to map `haiku` requests to. Defaults to `gpt-4.1-mini`. Ignored when `PREFERRED_PROVIDER=anthropic`.
   *   `OLLAMA_BASE_URL` (Optional): Ollama server URL. Defaults to `http://localhost:11434`.
   *   `CUSTOM_BASE_URL` (Optional): Custom/corporate API endpoint URL.
   *   `CUSTOM_API_KEY` (Optional): API key for custom/corporate endpoint.

   **Mapping Logic:**
   - If `PREFERRED_PROVIDER=openai` (default), `haiku`/`sonnet` map to `SMALL_MODEL`/`BIG_MODEL` prefixed with `openai/`.
   - If `PREFERRED_PROVIDER=google`, `haiku`/`sonnet` map to `SMALL_MODEL`/`BIG_MODEL` prefixed with `gemini/` *if* those models are in the server's known `GEMINI_MODELS` list (otherwise falls back to OpenAI mapping).
   - If `PREFERRED_PROVIDER=anthropic`, `haiku`/`sonnet` requests are passed directly to Anthropic with the `anthropic/` prefix without remapping to different models.
   - If `PREFERRED_PROVIDER=ollama`, `haiku`/`sonnet` map to `SMALL_MODEL`/`BIG_MODEL` and are routed to local Ollama. **No fallback to public APIs.**
   - If `PREFERRED_PROVIDER=custom`, `haiku`/`sonnet` map to `SMALL_MODEL`/`BIG_MODEL` and are routed to `CUSTOM_BASE_URL`. **No fallback to public APIs.**
   - If `PREFERRED_PROVIDER=openai` with a custom `OPENAI_BASE_URL` (not api.openai.com), **no fallback to public APIs** occurs.

4. **Run the server**:
   ```bash
   uv run uvicorn server:app --host 0.0.0.0 --port 8082 --reload
   ```
   *(`--reload` is optional, for development)*

#### Docker

If using docker, download the example environment file to `.env` and edit it as described above.
```bash
curl -O .env https://raw.githubusercontent.com/1rgs/claude-code-proxy/refs/heads/main/.env.example
```

Then, you can either start the container with [docker compose](https://docs.docker.com/compose/) (preferred):

```yml
services:
  proxy:
    image: ghcr.io/1rgs/claude-code-proxy:latest
    restart: unless-stopped
    env_file: .env
    ports:
      - 8082:8082
```

Or with a command:

```bash
docker run -d --env-file .env -p 8082:8082 ghcr.io/1rgs/claude-code-proxy:latest
```

### Using with Claude Code 🎮

1. **Install Claude Code** (if you haven't already):
   ```bash
   npm install -g @anthropic-ai/claude-code
   ```

2. **Connect to your proxy**:
   ```bash
   ANTHROPIC_BASE_URL=http://localhost:8082 claude
   ```

3. **That's it!** Your Claude Code client will now use the configured backend models (defaulting to Gemini) through the proxy. 🎯

## Model Mapping 🗺️

The proxy automatically maps Claude models to either OpenAI or Gemini models based on the configured model:

| Claude Model | Default Mapping | When BIG_MODEL/SMALL_MODEL is a Gemini model |
|--------------|--------------|---------------------------|
| haiku | openai/gpt-4o-mini | gemini/[model-name] |
| sonnet | openai/gpt-4o | gemini/[model-name] |

### Supported Models

#### OpenAI Models
The following OpenAI models are supported with automatic `openai/` prefix handling:
- o3-mini
- o1
- o1-mini
- o1-pro
- gpt-4.5-preview
- gpt-4o
- gpt-4o-audio-preview
- chatgpt-4o-latest
- gpt-4o-mini
- gpt-4o-mini-audio-preview
- gpt-4.1
- gpt-4.1-mini

#### Gemini Models
The following Gemini models are supported with automatic `gemini/` prefix handling:
- gemini-2.5-pro
- gemini-2.5-flash

### Model Prefix Handling
The proxy automatically adds the appropriate prefix to model names:
- OpenAI models get the `openai/` prefix
- Gemini models get the `gemini/` prefix
- The BIG_MODEL and SMALL_MODEL will get the appropriate prefix based on whether they're in the OpenAI or Gemini model lists

For example:
- `gpt-4o` becomes `openai/gpt-4o`
- `gemini-2.5-pro-preview-03-25` becomes `gemini/gemini-2.5-pro-preview-03-25`
- When BIG_MODEL is set to a Gemini model, Claude Sonnet will map to `gemini/[model-name]`

### Customizing Model Mapping

Control the mapping using environment variables in your `.env` file or directly:

**Example 1: Default (Use OpenAI)**
No changes needed in `.env` beyond API keys, or ensure:
```dotenv
OPENAI_API_KEY="your-openai-key"
GEMINI_API_KEY="your-google-key" # Needed if PREFERRED_PROVIDER=google
# PREFERRED_PROVIDER="openai" # Optional, it's the default
# BIG_MODEL="gpt-4.1" # Optional, it's the default
# SMALL_MODEL="gpt-4.1-mini" # Optional, it's the default
```

**Example 2a: Prefer Google (using GEMINI_API_KEY)**
```dotenv
GEMINI_API_KEY="your-google-key"
OPENAI_API_KEY="your-openai-key" # Needed for fallback
PREFERRED_PROVIDER="google"
# BIG_MODEL="gemini-2.5-pro" # Optional, it's the default for Google pref
# SMALL_MODEL="gemini-2.5-flash" # Optional, it's the default for Google pref
```

**Example 2b: Prefer Google (using Vertex AI with Application Default Credentials)**
```dotenv
OPENAI_API_KEY="your-openai-key" # Needed for fallback
PREFERRED_PROVIDER="google"
VERTEX_PROJECT="your-gcp-project-id"
VERTEX_LOCATION="us-central1"
USE_VERTEX_AUTH=true
# BIG_MODEL="gemini-2.5-pro" # Optional, it's the default for Google pref
# SMALL_MODEL="gemini-2.5-flash" # Optional, it's the default for Google pref
```

**Example 3: Use Direct Anthropic ("Just an Anthropic Proxy" Mode)**
```dotenv
ANTHROPIC_API_KEY="sk-ant-..."
PREFERRED_PROVIDER="anthropic"
# BIG_MODEL and SMALL_MODEL are ignored in this mode
# haiku/sonnet requests are passed directly to Anthropic models
```

*Use case: This mode enables you to use the proxy infrastructure (for logging, middleware, request/response processing, etc.) while still using actual Anthropic models rather than being forced to remap to OpenAI or Gemini.*

**Example 4: Use Specific OpenAI Models**
```dotenv
OPENAI_API_KEY="your-openai-key"
GEMINI_API_KEY="your-google-key"
PREFERRED_PROVIDER="openai"
BIG_MODEL="gpt-4o" # Example specific model
SMALL_MODEL="gpt-4o-mini" # Example specific model
```

**Example 5: Use Local Ollama Models** 🦙
```dotenv
PREFERRED_PROVIDER="ollama"
OLLAMA_BASE_URL="http://localhost:11434"
BIG_MODEL="llama3.3:70b"
SMALL_MODEL="llama3.2:latest"
# No API keys needed - runs entirely locally
# No fallback to public APIs
```

*Use case: Run completely offline with local models. All requests go to your Ollama instance without any external API calls.*

**Example 6: Use Corporate/Private API** 🏢
```dotenv
PREFERRED_PROVIDER="custom"
CUSTOM_BASE_URL="https://your-corporate-api.example.com/v1"
CUSTOM_API_KEY="your-corporate-api-key"
BIG_MODEL="gpt-4-corporate"
SMALL_MODEL="gpt-3.5-corporate"
# No fallback to public APIs
```

*Use case: Route all requests through your corporate API gateway without any calls to public APIs like OpenAI or Gemini.*

**Example 7: Use Azure OpenAI**
```dotenv
PREFERRED_PROVIDER="openai"
OPENAI_BASE_URL="https://your-resource.openai.azure.com/openai/deployments/your-deployment"
OPENAI_API_KEY="your-azure-api-key"
BIG_MODEL="gpt-4"
SMALL_MODEL="gpt-35-turbo"
# No fallback to public OpenAI API (custom base URL detected)
```

*Use case: Use Azure OpenAI Service instead of public OpenAI API. The proxy detects the custom base URL and won't fall back to public APIs.*

## How It Works 🧩

This proxy works by:

1. **Receiving requests** in Anthropic's API format 📥
2. **Mapping models** based on your `PREFERRED_PROVIDER` configuration 🗺️
3. **Translating** the requests to the target format via LiteLLM 🔄
4. **Sending** the translated request to your configured backend (OpenAI, Gemini, Ollama, custom API, or Anthropic) 📤
5. **Converting** the response back to Anthropic format 🔄
6. **Returning** the formatted response to the client ✅

The proxy handles both streaming and non-streaming responses, maintaining compatibility with all Claude clients. 🌊

### No Public API Fallback Mode

When using `ollama`, `custom`, or a custom `OPENAI_BASE_URL`, the proxy operates in "no fallback" mode:
- All requests are routed exclusively to your configured endpoint
- No requests are ever sent to public APIs (OpenAI, Gemini, Anthropic)
- This ensures complete control over where your data is sent

## Contributing 🤝

Contributions are welcome! Please feel free to submit a Pull Request. 🎁
