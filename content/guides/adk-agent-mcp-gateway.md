---
title: Build your first ADK agent with Docker MCP Gateway
keywords: adk, agent development kit, mcp, docker, gemini, ai agent, model context protocol
summary: |
  Create an AI agent using the Agent Development Kit (ADK) that connects to 300+ containerized tools through Docker's MCP Gateway.
params:
  time: 25 minutes
  tags: [AI]
---

## Introduction

The Agent Development Kit (ADK) is a framework for building AI agents in Python or TypeScript. These agents don't respond to prompts—they execute tasks autonomously by breaking down goals, calling tools, and handling multi-step workflows.

Docker's MCP Gateway turns ADK agents into production-grade tools. Instead of managing tool installations, dependencies, and credentials across your team, the Gateway runs MCP servers as isolated containers. Your agent connects to one endpoint and gets access to hundreds of verified tools: GitHub, databases, cloud APIs, search engines, and more.

This guide shows you how to build a research agent that uses the DuckDuckGo and GitHub MCP servers to find information and create issues automatically. You'll see how ADK's agent loop works, how to connect it to Docker's MCP infrastructure, and how to run everything locally with Gemini.

![ADK agent architecture with MCP Gateway](./images/adk-mcp-architecture.svg)

## Prerequisites

- [Docker Desktop 4.43 or later](../get-started/get-docker.md)
- [Python 3.10 or later](https://www.python.org/downloads/)
- Google AI API key from [aistudio.google.com/apikey](https://aistudio.google.com/apikey)
- Basic understanding of Python and command-line tools

## Step 1: Set up your development environment

Create a new directory for your agent and set up a Python virtual environment.

```console
$ mkdir adk-research-agent
$ cd adk-research-agent
$ python -m venv .venv
$ source .venv/bin/activate  # On Windows: .venv\Scripts\activate
```

Install the ADK package and required dependencies.

```console
$ pip install agent-dev-kit python-dotenv
```

Create a `.env` file to store your API credentials.

```console
$ cat > .env << EOF
GOOGLE_API_KEY=your_api_key_here
EOF
```

Replace `your_api_key_here` with your actual API key from Google AI Studio.

## Step 2: Configure Docker MCP Gateway

The MCP Gateway manages MCP servers as Docker containers. You'll configure it to run the DuckDuckGo and GitHub servers that your agent will use.

Create a `docker-compose.yml` file in your project directory.

```yaml
services:
  mcp-gateway:
    image: docker/mcp-gateway:latest
    ports:
      - "8811:8811"
    command:
      - --transport=sse
      - --servers=duckduckgo,github
    environment:
      - GITHUB_TOKEN=${GITHUB_TOKEN}
```

If you want to use the GitHub MCP server, add your GitHub token to the `.env` file.

```console
$ echo "GITHUB_TOKEN=your_github_token" >> .env
```

Start the MCP Gateway.

```console
$ docker compose up -d
```

Verify the Gateway is running and has loaded both servers.

```console
$ docker compose logs mcp-gateway
```

You should see log entries indicating that the DuckDuckGo and GitHub MCP servers started successfully.

## Step 3: Create your first ADK agent

Create a file named `agent.py` with your agent implementation.

```python
import os
from dotenv import load_dotenv
from adk import Agent, MCPTool

# Load environment variables
load_dotenv()

# Configure the agent with Gemini
agent = Agent(
    model="gemini-2.5-flash",
    api_key=os.getenv("GOOGLE_API_KEY"),
    system_prompt="""You are a research assistant that helps users find
    information and track it in GitHub issues. When asked to research a topic:

    1. Search for relevant information using DuckDuckGo
    2. Synthesize the findings into a clear summary
    3. If the user wants, create a GitHub issue with your findings

    Be concise and cite sources when possible."""
)

# Connect to Docker MCP Gateway
mcp_endpoint = "http://localhost:8811/sse"
agent.add_tool(MCPTool(endpoint=mcp_endpoint))

# Run the agent
if __name__ == "__main__":
    print("Research Agent ready. Ask me to research something!")
    print("Example: 'Research the latest features in Docker 27 and create a GitHub issue'\n")

    while True:
        user_input = input("You: ")
        if user_input.lower() in ["exit", "quit"]:
            break

        response = agent.run(user_input)
        print(f"\nAgent: {response}\n")
```

This agent connects to your MCP Gateway and automatically discovers all available tools from the DuckDuckGo and GitHub servers.

## Step 4: Run your agent

Execute your agent and test it with a research query.

```console
$ python agent.py
```

Try asking your agent to research something.

```text
You: Research the latest Docker AI features announced in 2026

Agent: I'll search for information about Docker's recent AI features.

[The agent calls the DuckDuckGo search tool through MCP Gateway]

Based on my research, Docker announced several AI features in 2026:

1. MCP Gateway - A centralized orchestrator for Model Context Protocol servers
2. Docker Model Runner improvements with extended model support
3. Enhanced Sandboxes with E2B integration for 200+ tools
4. Scout security scanning integration with AI-powered vulnerability analysis

Sources:
- docs.docker.com
- docker.com/blog
```

## Step 5: Test GitHub integration

If you configured the GitHub token, test creating an issue.

```text
You: Create a GitHub issue in my-org/my-repo summarizing the Docker AI features we discussed

Agent: I'll create an issue with the research findings.

[The agent calls the GitHub MCP server to create an issue]

Created issue #42: "Docker AI Features 2026 Summary"
https://github.com/my-org/my-repo/issues/42
```

The agent uses the GitHub MCP server running in the Gateway to authenticate and create the issue. You don't need to handle GitHub's API directly—the MCP server manages authentication, rate limiting, and API calls.

## Step 6: Add more MCP servers

The Docker MCP Catalog has over 300 verified servers. Add more capabilities to your agent by updating the Gateway configuration.

```yaml
services:
  mcp-gateway:
    image: docker/mcp-gateway:latest
    ports:
      - "8811:8811"
    command:
      - --transport=sse
      - --servers=duckduckgo,github,postgres,grafana
    environment:
      - GITHUB_TOKEN=${GITHUB_TOKEN}
      - POSTGRES_CONNECTION_STRING=${POSTGRES_CONNECTION_STRING}
      - GRAFANA_URL=${GRAFANA_URL}
      - GRAFANA_TOKEN=${GRAFANA_TOKEN}
```

Restart the Gateway to load the new servers.

```console
$ docker compose restart mcp-gateway
```

Your agent automatically gets access to PostgreSQL and Grafana tools without code changes. Update the system prompt to guide the agent on when to use these new capabilities.

## Understanding the architecture

When your agent runs a task, this is what happens:

1. **Agent receives goal**: ADK's agent loop takes your input and the system prompt
2. **Agent plans**: The model decides which tools to call and in what order
3. **Tool discovery**: ADK queries the MCP Gateway for available tools
4. **Tool execution**: The Gateway routes tool calls to the appropriate MCP server container
5. **Server processes request**: The containerized MCP server (DuckDuckGo, GitHub, etc.) executes the action
6. **Results return**: Data flows back through Gateway → ADK → your agent's response

The Gateway handles:

- Container lifecycle (starting, stopping servers as needed)
- Credential injection (your tokens never touch the agent code)
- Security isolation (each MCP server runs in its own container)
- Request routing (matching tool calls to the right server)

## Improving your agent

### Use prompt caching for efficiency

ADK supports prompt caching, which reduces costs when you're making repeated calls with similar context.

```python
agent = Agent(
    model="claude-sonnet-4-5",
    api_key=os.getenv("ANTHROPIC_API_KEY"),
    system_prompt="""...""",
    enable_prompt_caching=True
)
```

This feature is available with Claude models and reduces token costs when your agent processes multiple requests with the same system prompt and tool definitions.

### Add streaming for better UX

Stream responses as they're generated instead of waiting for completion.

```python
for chunk in agent.stream(user_input):
    print(chunk, end="", flush=True)
print()  # newline when done
```

### Implement conversation memory

Track conversation history to build multi-turn interactions.

```python
conversation_history = []

while True:
    user_input = input("You: ")
    if user_input.lower() in ["exit", "quit"]:
        break

    # Add user message to history
    conversation_history.append({"role": "user", "content": user_input})

    # Run agent with full history
    response = agent.run(conversation_history)

    # Add agent response to history
    conversation_history.append({"role": "assistant", "content": response})

    print(f"\nAgent: {response}\n")
```

## Debugging MCP connections

If your agent can't connect to the Gateway or tools aren't working:

Check that the Gateway is running and healthy.

```console
$ docker compose ps
```

View Gateway logs to see tool registration and requests.

```console
$ docker compose logs -f mcp-gateway
```

List available tools by querying the Gateway directly.

```console
$ curl http://localhost:8811/tools
```

Verify your environment variables are set correctly.

```console
$ docker compose config
```

## What's next

- Explore the [Docker MCP Catalog](https://hub.docker.com/catalogs/mcp/) to find more tools for your agent
- Learn about [running ADK agents in Docker Sandboxes](../manuals/ai/sandboxes/_index.md) for secure execution
- Read the [ADK documentation](https://adk.dev/docs/) for advanced agent patterns
- Deploy your agent to production using [Docker Compose in production](../manuals/compose/how-tos/production.md)
- Check out the complete [ADK + MCP example repository](https://github.com/docker/compose-for-agents) for more patterns

Need help with your agent or MCP Gateway setup? Visit the [Docker Community Forums](https://forums.docker.com) or explore the [MCP Gateway troubleshooting guide](../manuals/ai/mcp-catalog-and-toolkit/mcp-gateway.md).
