> [!NOTE]  
> The MCP server is currently available under `https://gemini-mcp-server-231532712093.europe-west1.run.app/mcp/`. It is deployed to Google Cloud Run and can be authenticated using an AI Studio API key. see [examples/test_remote.py](examples/test_remote.py) for an example on how to use the server with the `google-genai` client.

# Gemini MCP Server

A Model Context Protocol server that provides access to Google's Gemini API. This server enables LLMs to perform intelligent web searches, generate content, and access other Gemini features. It supports both STDIO and streamable-http transport modes and can be run locally or remotely. If you use STDIO mode it will try to use the `GEMINI_API_KEY` environment variable. If you use streamable-http mode it will try to use the Bearer token in the Authorization header.

**Available Tools:**
- **web_search** - Performs a web search using Gemini and returns synthesized results with citations
  - `query` (string, required): The search query to execute
  - `include_citations` (boolean, optional): Whether to include citations in the response. Default is `False`.
- **use_gemini** - Delegates a task to a specified Gemini 2.5 model (Pro or Flash).
  - `prompt` (string, required): The prompt or task for Gemini.
  - `model` (string, optional): The Gemini model to use. Default is `gemini-2.5-flash-preview-05-20`.


## Installation

```bash
pip install git+https://github.com/philschmid/gemini-mcp-server.git
```

## Authentication

- **STDIO mode**: Uses `GEMINI_API_KEY` environment variable
- **HTTP mode**: Requires Bearer token in Authorization header

### Running the Server

#### STDIO Mode (Local/Direct Integration)

```bash
GEMINI_API_KEY="your_gemini_api_key_here" gemini-mcp --transport stdio
```

#### HTTP Mode (Network Access)

```bash
gemini-mcp --transport streamable-http
```

The server will start on `http://0.0.0.0:8000/mcp/`

## Deployment

You can deploy the Gemini MCP Server as Remote MCP Server to Google Cloud Run to make it available easily available to any client. 

To deploy the server, run the following command from your terminal, replacing `[PROJECT-ID]` and `[REGION]` with your Google Cloud project ID and desired region:

```bash
# Set your project ID and region
export PROJECT_ID=remote-mcp-test-462811
export REGION=europe-west1
export SERVICE_NAME=gemini-mcp-server

# Authenticate with Google Cloud
gcloud auth login
gcloud config set project $PROJECT_ID

# Enable required services
gcloud services enable run.googleapis.com artifactregistry.googleapis.com cloudbuild.googleapis.com

# Deploy the service
gcloud run deploy $SERVICE_NAME \
  --source . \
  --region $REGION \
  --port 8000 \
  --allow-unauthenticated
```

The command will build the Docker image, push it to Google Artifact Registry, and deploy it to Cloud Run. After the deployment is complete, you will get a URL for your service. We will allow unauthenticated access to the service this means that anyone with the URL can send requests to the server, which it self is protected by an Authorization header. If you want to secure the service you can follow the instructions in the [Cloud Run documentation](https://cloud.google.com/run/docs/authenticating/service-to-service).

**cleanup**

```bash
SERVICE_NAME=gemini-mcp-server
REGION=europe-west1
gcloud run services delete $SERVICE_NAME --region $REGION
```

## Usage Examples

Add to your `mcpServers` configuration:

**STDIO Mode:**
```json
{
  "mcpServers": {
    "gemini-search": {
      "command": "gemini-mcp",
      "args": ["--transport", "stdio"],
      "env": {
        "GEMINI_API_KEY": "your_gemini_api_key_here"
      }
    }
  }
}
```

**HTTP Mode:**
```json
{
  "mcpServers": {
    "gemini-mcp": {
      "url": "https://remote-mcp-test.com/mcp/", // replace with your remote mcp server url
      "headers": { "Authorization": "Bearer YOUR_KEY" } // replace with your AI Studio API key
    }
  }
}
```

or check out the example in the [examples/test_remote.py](examples/test_remote.py) file.

```python
from mcp.client.streamable_http import streamablehttp_client

remote_url = "https://remote-mcp-test.com/mcp/" # replace with your remote mcp server url

async with streamablehttp_client(
    remote_url, headers={"Authorization": f"Bearer {api_key}"}
) as (read, write, _):
```

### With MCP Inspector

Start the server with streamable-http and test your server using the MCP inspector. Alternatively start inspector and run the server with stdio.

```bash
npx @modelcontextprotocol/inspector
```

## Web Search Tool Example

With `include_citations` set to `False`:

```json
{
  "text": "Recent advancements in AI include breakthrough developments in large language models, computer vision, and autonomous systems..."
}
```

With `include_citations` set to `True`:

```json
{
  "text": "Recent advancements in AI include breakthrough developments in large language models, computer vision, and autonomous systems...",
  "web_search_queries": ["latest AI developments 2024", "AI breakthroughs"],
  "citations": [
    {
      "start_index": 24,
      "end_index": 56,
      "sources": [
        {
          "title": "Latest AI Developments 2024",
          "uri": "https://example.com/ai-news"
        }
        ...
      ],
      "text": "breakthrough developments in large language models"
    },
    ...
  ]
}
```

## Use Gemini Tool Response Example

```json
{
  "text": "The capital of France is Paris."
}
```

## Testing

To run the tests, run the following command from the root directory:

Note: You need to set the `GEMINI_API_KEY` environment variable to run the tests.

```bash
pytest
```

## License

This project is licensed under the MIT License.