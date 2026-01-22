---
layout: post
title: "LangGraph Agents: Deploy as Hosted Agents in Microsoft Foundry"
date: 2025-12-25
author: Srikantan Sankaran
tags:
  - Azure
  - AI Agents
  - LangGraph
  - Microsoft Foundry
  - Hosted Agents
  - Azure AI
  - Multi-Agent Systems
  - Microsoft Teams
  - M365 Copilot
description: "Learn how to take your locally developed LangGraph AI agent and deploy it as an enterprise-ready, production-grade hosted agent with turnkey publishing to Microsoft Teams and Microsoft 365 Copilot using Microsoft Foundry."
image: "/blog/images/banner_image1.png"
---

## Introduction: The AI Agent Era and Microsoft Foundry

At Microsoft Ignite 2025, Microsoft unveiled a transformative platform that's reshaping how developers build, deploy, and manage AI agents: **Microsoft Foundry** (formerly known as Azure AI Foundry).

Rather than forcing developers into a single framework or development approach, Foundry embraces the reality that teams have different preferences, existing investments, and specialized requirements. The platform provides a comprehensive suite of capabilities:

### 🚀 Key Capabilities of Microsoft Foundry

| Capability | Description |
|------------|-------------|
| **Framework Freedom** | Build agents using your preferred framework—LangGraph, Microsoft Agent Framework, etc. |
| **Microsoft Agent Framework** | Leverage Microsoft's agent development tools to package and deploy your agents |
| **Managed Hosting** | Deploy agents as containerized, auto-scaling endpoints on Azure Container Apps |
| **Enterprise Security** | Automatic Entra Identity assignment for secure, authenticated access |
| **Turnkey Telemetry** | Built-in observability with tracing, logging, and performance monitoring |
| **One-Click Publishing** | Publish agents directly to Microsoft Teams and Microsoft 365 Copilot with no additional code |
| **High Availability** | Automatic failover and scaling without infrastructure management |

> 📢 **Note:** Microsoft Foundry is currently in Public Preview at the time of this writing.

### 🎥 Watch the Video Walkthrough

[![Watch the video](https://img.youtube.com/vi/8PezEmkvhiY/maxresdefault.jpg)](https://youtu.be/8PezEmkvhiY)

## What We're Building: A Multi-Agent Travel Support System

In this article, we'll walk through a practical example: taking a LangGraph-based multi-agent travel support system and deploying it as a Hosted Agent on Microsoft Foundry.

### The Travel Agent Architecture

The travel agent implements a sophisticated **supervisor pattern** where a primary assistant intelligently routes customer requests to specialized sub-agents. This sample is based on the popular [LangGraph Customer Support Bot tutorial](https://langchain-ai.github.io/langgraph/tutorials/customer-support/customer-support/), demonstrating complex workflows with conditional routing, state management across conversation turns, and graceful escalation.

![Multi Agent - Travel Agent](/assets/images/posts/2025-12-25-langgraph-foundry/image1.png)

| Specialist Agent | Responsibilities |
|------------------|------------------|
| ✈️ **Flight Assistant** | Search flights, update tickets, cancel bookings |
| 🏨 **Hotel Assistant** | Find hotels, make reservations, modify bookings |
| 🚗 **Car Rental Assistant** | Search rentals, book vehicles, update reservations |
| 🎯 **Excursions Assistant** | Find activities, book tours, manage trip recommendations |


## From Local Development to Enterprise Deployment

Here's the high-level journey we'll take:

![Local Dev To Cloud Deployment](/assets/images/posts/2025-12-25-langgraph-foundry/image2.png)

## The Implementation Journey

The code base from the original LangGraph sample was taken and additional wrappers required to host that in Microsoft Foundry as a Hosted Agent were added. See table below

![Server-Side File Overview](/assets/images/posts/2025-12-25-langgraph-foundry/image3.png)

### Key Files Overview

- **`workflow_core.py`** - Exposes LangGraph Agent as an Agent Framework compatible Agent
- **`custom_state_converter.py`** - Handles streaming response consolidation for Teams/M365
- **`container.py`** - Container entrypoint for environment and observability setup

### Exposing LangGraph as Agent Framework Compatible

`workflow_core.py` exposes the LangGraph Agent as an Agent Framework compatible Agent:

```python
from __future__ import annotations

import os
import logging

from dotenv import load_dotenv
from azure.ai.agentserver.langgraph import from_langgraph
from azure.identity.aio import DefaultAzureCredential, ManagedIdentityCredential

# Set up logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Load .env here as well to cover direct imports of this module
load_dotenv(override=True)

from travel_agent.app import part_4_graph
from custom_state_converter import RobustStateConverter

def create_agent(chat_client=None, as_agent: bool = True):
    """Expose the LangGraph as an Agent Framework-compatible agent.

    The chat_client parameter is accepted for parity with the agent-framework sample,
    but is not required by the LangGraph adapter.
    
    Uses the travel agent graph directly - no wrapper needed.
    Uses RobustStateConverter to fix non-streaming response issues with tool calls.
    """
    logger.info("create_agent: Using part_4_graph with RobustStateConverter")
    adapter = from_langgraph(part_4_graph, state_converter=RobustStateConverter())
    return adapter.as_agent() if as_agent else adapter
```

### Custom State Converter for Non-Streaming Clients

`custom_state_converter.py` is required to handle the scenario where clients like Microsoft Teams and Microsoft 365 Chat Agents need to consume the streaming response from LangGraph as a single consolidated response:

```python
def state_to_response(self, state: Any, context: AgentRunContext) -> Response:
    """
    Convert the final LangGraph state to a Response.
    
    With stream_mode="values", the state is the final graph state dict,
    not a list of step updates. We extract the last AI message as the response.
    """
    output = []
    
    # With stream_mode="values", state is the final state dict
    messages = state.get("messages", [])
    
    # Find the last AI message that has content (the actual response to user)
    last_ai_message = None
    for msg in reversed(messages):
        if isinstance(msg, AIMessage):
            # Skip AI messages that are only tool calls (no text content)
            if msg.content and not msg.tool_calls:
                last_ai_message = msg
                break
    
    if last_ai_message:
        # Convert to output format
        content_text = last_ai_message.content
        if isinstance(content_text, list):
            content_text = " ".join(
                item.get("text", "") if isinstance(item, dict) else str(item)
                for item in content_text
            )
        
        output.append(
            project_models.ResponsesAssistantMessageItemResource(
                content=[
                    project_models.ItemContent({
                        "type": project_models.ItemContentType.OUTPUT_TEXT,
                        "text": content_text,
                        "annotations": [],
                    })
                ],
                id=context.id_generator.generate_message_id(),
                status="completed",
            )
        )
    # ... rest of the implementation
```

## Step-by-Step Deployment Guide

### Step 1: Local Development and Testing

The agent was developed and tested locally using VS Code with GitHub Copilot Agent Mode. The local playground allows rapid iteration before deployment.

![local playground 01](https://raw.githubusercontent.com/MSFT-Innovation-Hub-India/LangGraph-Foundry-HostedAgent-TravelAgent/main/images/streaming-responses.png)
* Figure: Testing the agent locally before deployment*


### Step 2: Testing in Local VS Code Microsoft Foundry Playground

To host a LangGraph agent on Foundry, you need:

- A **container entrypoint** (`container.py`) that sets up the environment and observability
- A **workflow adapter** (`workflow_core.py`) that wraps your LangGraph with the Agent Framework adapter

![Local Playground Testing](https://raw.githubusercontent.com/MSFT-Innovation-Hub-India/LangGraph-Foundry-HostedAgent-TravelAgent/main/images/LocalPlayground.png)
*Figure: Testing the agent locally in VS Code before deployment*

### Step 3: Deploying as a Hosted Agent

Using the VS Code workflow for hosted agents, the deployment process is streamlined through the Azure AI Foundry extension.

![Deploy as Hosted Agent](https://raw.githubusercontent.com/MSFT-Innovation-Hub-India/LangGraph-Foundry-HostedAgent-TravelAgent/main/images/DeployHostedAgent.png)
*Deploying the LangGraph agent to Microsoft Foundry via VS Code*

### Step 4: Testing in Foundry Playground

Once deployed, test your agent in the Foundry Playground to verify all functionality works as expected in the cloud environment.

![Foundry Playground](https://raw.githubusercontent.com/MSFT-Innovation-Hub-India/LangGraph-Foundry-HostedAgent-TravelAgent/main/images/Playground-Agent.png)
*Testing the deployed agent in Microsoft Foundry's Playground*

### Step 5: Publishing to Enterprise Channels

With the agent running successfully, publishing to Teams and M365 Copilot is just a few clicks away through the Foundry portal!

![Access from Microsoft Teams](/assets/images/posts/2025-12-25-langgraph-foundry/msft-teams.png)
*Figure: The travel agent accessible in Microsoft Teams*

![Access from Microsoft 365 Copilot Chat](/assets/images/posts/2025-12-25-langgraph-foundry/m365-copilot-agent.png)
*Figure: The travel agent accessible in Microsoft 365 Copilot Chat*

## Important Considerations

During development of this sample, we encountered some challenges worth noting:

### State Management for Non-Streaming Clients

When publishing to channels like Teams and M365 Copilot (which use non-streaming mode), LangGraph agents that use tools require a custom state converter to properly handle the response format. We developed a `RobustStateConverter` to address this.

### Version Routing

When publishing new versions of an existing Hosted Agent Application, you may encounter routing issues. A workaround is to publish new versions with different application names.

> 📌 For detailed technical solutions to these challenges, please refer to the [GitHub repository](https://github.com/MSFT-Innovation-Hub-India/LangGraph-Foundry-HostedAgent-TravelAgent).


### Architecture

Here is the overall architcture of the solution.
![Architecture](/assets/images/posts/2025-12-25-langgraph-foundry/architecture.png)

## Sample Conversation Flow

Here's an example of the multi-agent system in action:

**User:** "Hi, I need help with my upcoming trip. My passenger ID is 3442 587242"

**Agent:** "Hello! I can see you have a flight from Basel (BSL) to Paris (CDG) scheduled for [date]. How can I assist you with your trip today?"

**User:** "I'd like to book a hotel in Paris near the Eiffel Tower"

**Agent (routing to Hotel Assistant):** "I found several excellent hotels near the Eiffel Tower for your dates. Here are the top options:
1. Hôtel Plaza Athénée - 5-star, 0.5 km from Eiffel Tower
2. Pullman Paris Tour Eiffel - 4-star, 0.3 km from Eiffel Tower ..."

**User:** "Also, can you recommend some activities to do in Paris?"

**Agent (routing to Excursions Assistant):** "Paris has wonderful activities! Based on your dates, here are some recommendations:
1. Eiffel Tower Summit Access - Skip-the-line tickets
2. Louvre Museum Guided Tour
3. Seine River Dinner Cruise ..."

## Getting Started

Ready to build your own Hosted Agent on Microsoft Foundry? Here's how to get started:

1. **Clone the repository:**
   ```bash
   git clone https://github.com/MSFT-Innovation-Hub-India/LangGraph-Foundry-HostedAgent-TravelAgent
   ```

2. **Set up your environment** using the provided `.env.example`

3. **Test locally** with `python test_local.py`

4. **Deploy to Foundry** following the step-by-step instructions in the README

5. **Publish to Teams and M365** with just a few clicks!

### 💡 Development with GitHub Copilot Agent Mode

Throughout this development process, **GitHub Copilot Agent Mode** in VS Code with Claude Opus 4.5 was used extensively. From writing the initial agent code, debugging issues, packaging the agent for Foundry, to troubleshooting deployment challenges—Copilot's agentic capabilities significantly accelerated development and helped navigate the complexities of multi-agent orchestration and cloud deployment.

## Conclusion

Microsoft Foundry represents a significant step forward in enterprise AI development. By combining the flexibility of framework choice (like LangGraph) with production-grade hosting, security, and one-click publishing capabilities, Foundry enables developers to focus on building great agents while the platform handles the operational complexity.

The ability to take a locally-developed multi-agent system and deploy it as an enterprise-ready service—accessible through Microsoft Teams and M365 Copilot—with minimal additional code is transformative. Combined with built-in observability and security, Foundry provides a compelling path from prototype to production.

## Resources

- **GitHub Repository:** [LangGraph-Foundry-HostedAgent-TravelAgent](https://github.com/MSFT-Innovation-Hub-India/LangGraph-Foundry-HostedAgent-TravelAgent)
- **Microsoft Foundry Documentation:** [Azure AI Foundry Docs](https://learn.microsoft.com/en-us/azure/ai-foundry/)
- **LangGraph Documentation:** [LangGraph Docs](https://langchain-ai.github.io/langgraph/)
- **VS Code Workflow for Hosted Agents:** [Pro-Code Workflow Guide](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/how-to/vs-code-agents-workflow-pro-code)

---

*This article was developed with extensive use of GitHub Copilot Agent Mode in VS Code, demonstrating the power of AI-assisted development for building and deploying AI agents.*

