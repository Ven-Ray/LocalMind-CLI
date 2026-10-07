# LocalMind CLI

CLI chat application with conversation memory, multi-provider support (LM Studio, Ollama, Anthropic Claude), append-only CSV logging, and keyword-based historical context retrieval.

Provider is selected via the `MODEL_PROVIDER` environment variable; defaults to LM Studio. The app creates `data/memory_log.csv` automatically if it doesn't exist.

## Features

- **LangChain conversation memory** using custom `SimpleChatHistory` class for active session history
- **Multiple LLM providers**: LM Studio (default), Ollama, Anthropic Claude
- **Provider selection** via `MODEL_PROVIDER` environment variable
- **Append-only CSV logging** of all conversations to `data/memory_log.csv`
- **Historical context retrieval** from CSV using lightweight keyword matching
- **Relevant follow-up questions** generated based on current and historical context
- **Two separate memory types**: active in-memory history + historical CSV context

## Project Structure

```text
langchain-memory-demo/
├── app.py              # Main CLI application
├── requirements.txt    # Python dependencies
├── .env.example        # Environment configuration template
├── .gitignore          # Git ignore rules
├── README.md           # This file
├── Note.md             # Implementation notes for project owner
├── tests/
│   └── test_app.py     # Unit tests (pytest)
└── data/
    └── .gitkeep        # Ensures data directory is tracked by Git
```

The application automatically creates `data/memory_log.csv` when it does not exist. The CSV file stores conversation history with columns: `timestamp`, `session_id`, `role`, `message`.

## Dependencies

| Package | Purpose |
|---------|---------|
| `langchain` | Core LangChain framework for building LLM applications |
| `langchain-core` | LangChain core abstractions (prompts, chains, runnables) |
| `langchain-openai` | OpenAI-compatible API integration (used for LM Studio) |
| `langchain-ollama` | Ollama local model server integration |
| `langchain-anthropic` | Anthropic Claude API integration |
| `python-dotenv` | Load environment variables from `.env` file |

## Setup Instructions

### macOS / Linux

```bash
# Create virtual environment
python3 -m venv .venv

# Activate virtual environment
source .venv/bin/activate

# Upgrade pip
python -m pip install --upgrade pip

# Install dependencies
pip install -r requirements.txt

# Copy example environment file
cp .env.example .env

# Edit .env with your configuration (see below)

# Run the application
python app.py
```

### Windows PowerShell

```powershell
# Create virtual environment
py -3 -m venv .venv

# Activate virtual environment
.\.venv\Scripts\Activate.ps1

# Upgrade pip
python -m pip install --upgrade pip

# Install dependencies
pip install -r requirements.txt

# Copy example environment file
Copy-Item .env.example .env

# Edit .env with your configuration (see below)

# Run the application
python app.py
```

## Environment Configuration

Edit `.env` to configure your LLM provider. The `MODEL_PROVIDER` variable selects which provider to use (case-insensitive). Supported values: `lmstudio`, `ollama`, `anthropic`. If not set, defaults to `lmstudio`.

### LM Studio (Default)

```env
MODEL_PROVIDER=lmstudio
LM_STUDIO_BASE_URL=http://localhost:1234/v1
LM_STUDIO_MODEL=your-loaded-model-identifier
```

**How it works**: LM Studio exposes an OpenAI-compatible API. The application uses LangChain's `ChatOpenAI` integration with the LM Studio server URL and a placeholder API key (`"lm-studio"`). This is suitable for LM Studio and other OpenAI-compatible local servers.

### Ollama

```env
MODEL_PROVIDER=ollama
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_MODEL=your-ollama-model-name
```

**How it works**: The application uses LangChain's `ChatOllama` integration to communicate with the Ollama server.

### Anthropic Claude

```env
MODEL_PROVIDER=anthropic
ANTHROPIC_API_KEY=your-anthropic-api-key
ANTHROPIC_MODEL=claude-3-5-sonnet-latest
```

**How it works**: The application uses LangChain's `ChatAnthropic` integration with your Anthropic API key.

## LM Studio Setup Guide

1. **Download and install LM Studio**: Visit [lmstudio.ai](https://lmstudio.ai) and download the installer for your platform.

2. **Download a model**: Open LM Studio, go to the "Models" tab (left sidebar), search for a model you want (e.g., `llama-3`, `mistral`), and click Download. Wait for the download to complete.

3. **Load the model**: In the "Chat" tab, select your downloaded model from the dropdown at the bottom of the screen. Click "Load Model".

4. **Start the server**: Go to the "Server" tab (left sidebar). Ensure the correct model is selected in the dropdown. Click "Start Server". The default URL is `http://localhost:1234/v1`.

5. **Find your model identifier**: In LM Studio, go to the "Models" tab and find your downloaded model. The identifier is shown below the model name (e.g., `llama-3-8b-instruct.Q4_K_M`). Use this exact identifier in your `.env` file as `LM_STUDIO_MODEL`.

## Ollama Setup Guide

1. **Install Ollama**: Visit [ollama.com](https://ollama.com) and download the installer for your platform. Follow the installation instructions.

2. **Download a model**: Open a terminal and run:
   ```bash
   ollama pull llama3
   ```
   Replace `llama3` with any available Ollama model name (see [Ollama library](https://ollama.com/library)).

3. **Start the server**: On most platforms, Ollama starts automatically when you install it. If not, run:
   ```bash
   ollama serve
   ```

4. **Configure .env**: Set `OLLAMA_MODEL` to the name of your installed model (e.g., `llama3`).

## Anthropic Claude Setup Guide

1. **Get an API key**: Sign up at [console.anthropic.com](https://console.anthropic.com) and create an API key from your account settings.

2. **Configure .env**: Set `ANTHROPIC_API_KEY` to your API key and choose a model name for `ANTHROPIC_MODEL` (e.g., `claude-3-5-sonnet-latest`, `claude-3-opus-latest`).

## Running the CLI

After setup, run:
```bash
python app.py
```

The application starts an interactive CLI session. Type your messages and press Enter. The assistant responds using the configured LLM provider.

### CLI Commands

| Command | Description |
|---------|-------------|
| `history` | Display active in-memory conversation history for this session |
| `clear` | Clear active in-memory history (CSV log is preserved) |
| `quit` or `exit` | Exit the application |

Commands are case-insensitive. Any other input is treated as a user message to send to the assistant.

The system maintains two separate memory types: **active in-memory history** (per-session, lost on exit) and **historical CSV context** (permanent, keyword-retrieved across sessions). Provider selection is driven by the `MODEL_PROVIDER` environment variable.

## How It Works

### LangChain Implementation

The application uses several key LangChain components:

- **`ChatPromptTemplate`**: Constructs prompts with system instructions, conversation history placeholder, historical context, and user input
- **`MessagesPlaceholder`**: Inserts active conversation history into the prompt at runtime
- **`SimpleChatHistory`**: Custom in-memory chat history class that stores active session messages (lost when application exits)
- **`store` dictionary**: Maps session IDs to their `SimpleChatHistory` instances

The conversation flow is:
1. User types a message in the CLI
2. Application retrieves relevant historical context from CSV using keyword matching
3. User message is logged to CSV before model request
4. Chain is invoked with user input, active history (via `store[session_id].messages`), and retrieved historical context
5. Model generates response
6. Assistant response is logged to CSV after successful completion
7. Response is displayed in CLI

### Active In-Memory History vs CSV Historical Context

The application maintains two separate types of memory:

1. **Active in-memory history** (`SimpleChatHistory`): Stores messages for the current session only. Provides conversation continuity within a single run. Lost when the application exits. Cleared with the `clear` command.

2. **Historical CSV context** (`data/memory_log.csv`): Append-only log of all conversations across all sessions. Never used to restore active memory. Used as a read-only source for retrieving relevant past information via keyword matching before each model request. Persists across application runs and is not affected by the `clear` command.

This separation ensures that:
- Active conversation flows naturally within a session
- Past conversations inform future questions without cluttering active history
- The CSV log grows over time as a permanent record
- Clearing active memory doesn't lose historical context

### Keyword-Based Historical Retrieval

Before each model request, the application searches the CSV log for relevant past entries using lightweight keyword matching:

1. **Tokenization**: Text is converted to lowercase and split into words (alphanumeric tokens)
2. **Stop word removal**: Common English stop words are filtered out
3. **Relevance scoring**: Each historical entry is scored by counting overlapping keywords with the current query
4. **Sorting**: Entries are sorted by relevance score (descending), then by timestamp (newest first for ties)
5. **Limiting**: Only the top N entries (default: 10) are returned

This approach is simple, fast, and requires no external services or embeddings. It's not semantic search - it matches literal keywords rather than meaning. However, it effectively retrieves relevant past conversations when users ask about similar topics using similar words.

### Automatic CSV Creation

The application automatically creates the `data/` directory and `data/memory_log.csv` file if they don't exist:

- On startup, before the CLI loop begins
- Defensively before any read or write operation (in case the file was manually deleted while running)

If the CSV file is missing or empty, it's recreated with the header row. Existing conversation data is never overwritten or deleted during normal operation. The application can recover from a missing CSV file without failing.

### Follow-Up Questions Using Relevant Context

The system prompt instructs the model to generate 1-3 relevant follow-up questions after answering each user question. These questions leverage:

- **Active conversation history**: To avoid asking about topics already discussed in this session
- **Retrieved historical context**: To make follow-ups more relevant based on past conversations and interests

The model is instructed to ask concise, useful questions that would improve its answer or explore related topics, while avoiding repetitive or unrelated questions. If no follow-up question is useful, it writes "None."

## Security Considerations

- **`.env` file**: Contains API keys and configuration. Never commit `.env` to Git (it's in `.gitignore`).
- **API keys**: Store securely. Rotate periodically if exposed.
- **CSV logs**: Contain conversation text including any sensitive information you share. The CSV is stored locally on your machine.
- **Local providers** (LM Studio, Ollama): Run on localhost; data doesn't leave your machine unless the model sends it elsewhere.
- **Remote providers** (Anthropic): Data is sent to their servers for processing.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `Connection refused` or timeout errors | Ensure your LLM server (LM Studio/Ollama) is running and accessible at the configured URL |
| Model not loaded error in LM Studio | Load a model in LM Studio before starting the application |
| Invalid model identifier | Check the exact model name/identifier in your provider's interface |
| Missing `.env` file | Copy `.env.example` to `.env` and configure it |
| Placeholder values still in `.env` | Replace all placeholder values with actual configuration |
| CSV file errors | The application recreates missing or empty CSV files automatically. Check file permissions if issues persist |
| Malformed CSV rows | The application skips malformed rows during retrieval; they don't crash the app |
| Incompatible endpoint | Ensure your provider supports the API format used by LangChain's integration |
