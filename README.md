# Tripo

> AI travel intelligence for turning a travel idea into a practical, reviewable itinerary.

Tripo is a full-stack travel-planning application built around a LangGraph multi-agent workflow. A user describes a trip in natural language, and Tripo coordinates specialist agents for flights, accommodation, weather, budget analysis, and itinerary design. Before a final response is produced, the draft itinerary pauses for human review.

The application is served by FastAPI and includes a responsive browser interface, persistent LangGraph threads backed by PostgreSQL, and Model Context Protocol (MCP) integrations for travel research.

## Highlights

- Natural-language trip planning through a polished web interface
- Supervisor agent that selects only the specialist agents needed for each request
- Input guardrail for unrelated, harmful, or illegal requests
- Flight and airline research through AviationStack MCP
- Hotel and destination research through Tavily MCP
- Current weather and forecast data through a local Weather MCP server backed by OpenWeather
- Budget feasibility analysis with approximate-cost warnings when live prices are unavailable
- Human-in-the-loop approval and revision workflow
- PostgreSQL-backed LangGraph checkpointing so paused plans can be resumed by `thread_id`
- Markdown rendering, clipboard copy, and PDF download in the browser
- Docker image for deployment with Uvicorn

## How It Works

```mermaid
flowchart LR
		U[Travel request] --> G[FastAPI]
		G --> S[Supervisor and guardrail]
		S --> F[Flight agent]
		S --> H[Hotel agent]
		S --> W[Weather agent]
		S --> B[Budget agent]
		F --> I[Itinerary agent]
		H --> I
		W --> I
		B --> I
		I --> R[Human review]
		R -->|Approve or request changes| X[Final response agent]
		X --> G
		S -->|Blocked| G
		F -.-> A[AviationStack MCP]
		H -.-> T[Tavily MCP]
		W -.-> O[OpenWeather MCP server]
		S -.-> L[Groq LLM]
		R -.-> P[(PostgreSQL checkpoints)]
```

The supervisor always includes the itinerary agent. Specialist agents are selected dynamically from the user's request, then their outputs are combined into a draft. LangGraph interrupts execution at the human approval step and resumes the same thread when the user approves or requests a revision.

## Project Structure

```text
.
├── app.py                       # FastAPI application and HTTP routes
├── backend.py                   # LangGraph state, agents, routing, and persistence
├── mcp_client.py                # MCP server configuration and tool wrappers
├── custom_weather_mcp_server.py # Local FastMCP server for OpenWeather
├── requirements.txt             # Python dependencies
├── Dockerfile                   # Production container definition
├── templates/
│   └── index.html               # Browser application shell
└── static/
		├── script.js                # Frontend API calls and interactions
		└── style.css                 # Frontend styling and responsive layout
```

## Prerequisites

- Python 3.11 or newer
- A PostgreSQL database reachable by the application
- A Groq API key
- A Tavily API key
- An AviationStack API key
- An OpenWeather API key
- `uvx` available on `PATH` for the AviationStack MCP server
- Git, if cloning the project

The default LLM configured by the application is Groq's `openai/gpt-oss-120b` model. Change the model in `backend.py` and `mcp_client.py` if your provider configuration requires a different model.

## Quick Start

### 1. Create and activate a virtual environment

Windows PowerShell:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

macOS or Linux:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### 2. Install dependencies

```bash
python -m pip install --upgrade pip
pip install -r requirements.txt
```

Make sure the `uvx` command is installed and available to the same user running Tripo. The AviationStack MCP client starts `aviationstack-mcp` through `uvx` on demand.

### 3. Configure environment variables

Create a `.env` file in the project root:

```dotenv
GROQ_API_KEY=your_groq_api_key
TAVILY_API_KEY=your_tavily_api_key
AVIATIONSTACK_API_KEY=your_aviationstack_api_key
OPENWEATHER_API_KEY=your_openweather_api_key
DATABASE_URL=postgresql://username:password@host:5432/database
```

`backend.py` automatically appends `sslmode=require` to `DATABASE_URL` when it is not already present. Use a database connection string that your provider supports.

### 4. Start the application

```bash
python app.py
```

Open [http://127.0.0.1:8000](http://127.0.0.1:8000) in a browser.

For development, the equivalent Uvicorn command is:

```bash
uvicorn app:app --reload --host 127.0.0.1 --port 8000
```

The health endpoint is available at [http://127.0.0.1:8000/health](http://127.0.0.1:8000/health).

## Configuration

| Variable                  | Required                 | Used by                           | Description                                                              |
| ------------------------- | ------------------------ | --------------------------------- | ------------------------------------------------------------------------ |
| `GROQ_API_KEY`          | Yes                      | `backend.py`, `mcp_client.py` | LLM access for guardrails, routing, analysis, and response generation    |
| `DATABASE_URL`          | Yes                      | `backend.py`                    | PostgreSQL connection used by`PostgresSaver` for LangGraph checkpoints |
| `TAVILY_API_KEY`        | Yes for hotel research   | `mcp_client.py`                 | Tavily MCP streamable HTTP connection                                    |
| `AVIATIONSTACK_API_KEY` | Yes for flight research  | `mcp_client.py`                 | Passed to the AviationStack MCP process                                  |
| `OPENWEATHER_API_KEY`   | Yes for weather research | `custom_weather_mcp_server.py`  | Current weather and forecast requests                                    |

Never commit `.env` files, API keys, database passwords, or generated secrets. Use a secret manager or deployment-platform environment settings in production. If a key has been exposed, revoke it and issue a replacement.

## Using the Web App

1. Enter a request such as `Plan a 7-day trip to Japan from Delhi focused on food, culture, and hidden gems.`
2. Tripo validates and routes the request to the relevant specialist agents.
3. Review the generated draft itinerary.
4. Select **Approve & generate final** or enter feedback and select **Revise with feedback**.
5. Copy the completed plan or download it as a PDF.

The browser stores the current LangGraph `thread_id` in local storage so the approval request can resume the correct paused workflow.

## API Reference

### `GET /`

Returns the web application.

### `GET /health`

Returns a basic service status response:

```json
{
	"status": "ok",
	"message": "TripMate AI API is running",
	"features": [
		"supervisor_agent",
		"input_guardrail",
		"human_in_the_loop"
	]
}
```

### `POST /api/travel`

Starts a new travel-planning workflow. `thread_id` is optional; when omitted, the server creates one.

Request:

```json
{
	"message": "Plan a 5-day trip to Dubai from Dhaka with hotels and sightseeing.",
	"thread_id": null
}
```

The response includes `success`, `thread_id`, the selected agents, extracted constraints, specialist results, and either a draft requiring approval or a completed answer.

Important response fields:

| Field                 | Meaning                                                       |
| --------------------- | ------------------------------------------------------------- |
| `thread_id`         | Identifier required to resume the workflow                    |
| `answer`            | Draft or final text suitable for display                      |
| `requires_approval` | Whether the workflow is paused for human review               |
| `itinerary`         | Draft itinerary returned at the approval step                 |
| `selected_agents`   | Agents chosen by the supervisor                               |
| `trip_constraints`  | Destination, origin, duration, budget, style, and preferences |
| `guardrail_allowed` | Whether the request passed the input guardrail                |

### `POST /api/travel/approve`

Resumes a paused workflow using the stored PostgreSQL checkpoint.

Approve a draft:

```json
{
	"thread_id": "user_0123456789abcdef",
	"approved": true,
	"feedback": ""
}
```

Request a revision:

```json
{
	"thread_id": "user_0123456789abcdef",
	"approved": false,
	"feedback": "Reduce hotel costs and add one free day near the end of the trip."
}
```

When `approved` is `false`, non-empty `feedback` is required.

## MCP Integrations

Tripo uses `MultiServerMCPClient` with isolated server initialization so a failure in one integration does not prevent the others from loading.

| MCP server    | Transport                                      | Purpose                                                    |
| ------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| Tavily        | Streamable HTTP                                | Hotel and destination web research                         |
| AviationStack | Local stdio via`uvx`                         | Airport and airline information                            |
| Weather MCP   | Local stdio via the current Python interpreter | Current weather and five forecast entries from OpenWeather |

To inspect the tools exposed by the configured servers, call `get_all_tools()` from a Python session after environment variables are configured.

You can also run the local weather server directly:

```bash
python custom_weather_mcp_server.py
```

The normal application starts this server as an MCP subprocess automatically when weather tools are first requested.

## Docker

Build and run the image:

```bash
docker build -t tripo .
docker run --rm -p 8000:8000 --env-file .env tripo
```

Then open [http://127.0.0.1:8000](http://127.0.0.1:8000). The container listens on `0.0.0.0:8000` and expects the same environment variables described above.

For a hosted deployment, provision PostgreSQL separately, add the environment variables through the hosting platform's secret configuration, and expose port `8000`.

## Troubleshooting

### `GROQ_API_KEY is missing`

Confirm that `.env` is in the project root and that the variable name is spelled exactly as shown. Restart the process after changing environment variables.

### `DATABASE_URL is missing` or checkpoint errors

Verify the PostgreSQL URL, network access, credentials, and SSL requirements. The application runs `PostgresSaver.setup()` during startup, so the database must be reachable before the API can serve requests.

### AviationStack tools do not load

Check that `uvx` is installed and available on `PATH`, then verify `AVIATIONSTACK_API_KEY`. The AviationStack MCP package is started dynamically rather than installed directly from `requirements.txt`.

### Weather requests fail

Verify `OPENWEATHER_API_KEY` and confirm that the destination can be resolved to a city by the LLM destination extractor. Weather and forecast failures are converted into fallback guidance, but the result should still be verified before travel.

### Live prices are missing

Flight and accommodation prices are presented as estimates or research when live booking data is unavailable. Tripo is a planning assistant, not a booking or payment system. Confirm prices, availability, visa rules, and travel advisories with official providers before booking.

## Development Notes

- `backend.py` contains the graph state, agent implementations, routing, checkpoint setup, and FastAPI-facing helpers.
- `mcp_client.py` owns external MCP configuration and lazy tool initialization.
- `app.py` is intentionally thin and handles request validation, response shaping, and static/template serving.
- The frontend uses CDN-hosted `marked` and `html2pdf.js` assets in `templates/index.html`.
- No automated test suite is currently included. Add endpoint and graph tests before making production workflow changes.

## Security and Privacy

- Keep all provider keys and database credentials out of source control.
- Treat generated itineraries as advisory content and verify time-sensitive information.
- Review data retention and access controls for the PostgreSQL instance used for checkpoints.
- Add authentication, rate limiting, request logging controls, and stricter CORS policy before exposing the service publicly.
- Do not send sensitive personal information in travel prompts unless the deployment's data handling has been reviewed.

## License

This project is distributed under the [MIT License](LICENSE).
