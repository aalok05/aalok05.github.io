---

layout:  post

title:  "Building Modular AI Agents: A Plugin Architecture"

categories:  AI

author:  "Aalok Singh"

---

## The Problem: Monolithic AI Systems

Most AI systems start simple but quickly become unwieldy monoliths. Whether you're building chatbots, RAG systems, or AI assistants, you've probably experienced this evolution:

**Day 1:** "Let's build a simple AI assistant that can answer questions about our docs"

**Day 30:** "Now it needs to search our database, generate reports, and handle customer queries"

**Day 90:** "The system is slow, hard to test, and any change breaks something else"

### Why This Happens

**Knowledge Base Coupling:** Everything gets thrown into one giant vector database

**Pipeline Rigidity:** One retrieval system tries to handle all content types

**Prompt Bloat:** A single "universal" prompt grows to handle every edge case

**Integration Mess:** One API endpoint doing everything becomes impossible to maintain

### The Real Cost

- **Development slowdown:** Teams can't work independently
- **Testing nightmare:** Need to spin up everything to test one feature
- **Deployment risk:** Any change affects the entire system
- **Scaling inefficiency:** Resource usage driven by your heaviest component

## The Solution: Plugin Architecture

Instead of one massive RAG solution, build modular agents that have unique capabilities:

```python
from abc import ABC, abstractmethod

class BaseModularAgent(ABC):
    """Base class for all Agents"""
    
    def __init__(self):
        self.agent = None
    
    @abstractmethod
    def get_metadata(self) -> dict:
        """Return metadata this agent"""
        return {
            "name": "agent_name",
            "description": "What this agent does"
        }
    
    @abstractmethod
    def get_prompt(self) -> str:
        """System prompt for the agent"""
        pass
    
    @abstractmethod
    async def initialize(self):
        """Initialize the agent"""
        # Create agent with the prompt
        pass

```

### Core Principles

**Single Responsibility:** Each plugin does one thing well

**Self-Contained:** Plugins manage their own resources and dependencies

**Discoverable:** System automatically finds and loads available plugin agents

**Composable:** Plugins can work together or independently

## Dynamic Discovery: Finding Your Plugin Agents

```python
async def find_and_initialize_agents(directory: str):
    """ Discover and initialize plugin agents"""
    agents = []
    
    for file in os.listdir(directory):
        if file.endswith('_pluginagent.py'):
            # Load the module
            module = importlib.import_module(file[:-3])
            
            # Find agent classes
            for item in dir(module):
                obj = getattr(module, item)
                if (isinstance(obj, type) and 
                    issubclass(obj, BaseModularAgent) and 
                    obj != BaseModularAgent):
                    # Create instance and initialize the agent
                    agents = obj()
                    await agents.initialize()
                    agents.append(agents)
    
    return agents
```

## Modular Agent Example

### Company Policy Agent

The below example omits the 'retrieval' part of RAG for simplicity. In real world this agent would most likely perform semantic search on a vector database which contains embeddings of company policy documents. It would define a function tool responsible for performing this search.

```python
from agents import Agent

class CompanyPolicyAgent(BaseModularAgent):
    def get_metadata(self) -> dict:
        return {
            "name": "document_search",
            "description": "Search through company documents and policies"
        }
    
    def get_prompt(self) -> str:
        return """You are a document search specialist. Help users find relevant 
        documents, policies, and information from the company knowledge base. 
        Always provide accurate, cited results."""
    
    async def initialize(self):
        metadata = self.get_metadata()
        self.agent = Agent(
                name=metadata["name"],
                instructions=self.get_prompt(),
                model="o3"
        )

```

## Putting It All Together with Agents SDK

```python
from agents import Agent, Runner
import os
import importlib

async def main():
    # Discover and initialize modular agents and use them as tools for orchestrator agent
    agentModules = {}
    
    for file in os.listdir('./plugin_agents'):
        if file.endswith('_pluginagent.py'):
            # Load the module
            module = importlib.import_module(file[:-3])
            
            # Find capability classes
            for item in dir(module):
                obj = getattr(module, item)
                if (isinstance(obj, type) and 
                    issubclass(obj, BaseModularAgent) and 
                    obj != BaseModularAgent):
                    # Create instance and initialize the agent
                    capability = obj()
                    await capability.initialize()
                    
                    metadata = capability.get_metadata()
                    agentModules[metadata["name"]] = capability
                    print(f"Loaded agent: {metadata['description']}")
    
    # Convert each plugin agent into a tool
    tools = []
    for name, agent in agentModules.items():
        metadata = agent.get_metadata()
        tool = agent.agent.as_tool(
            name=metadata["name"],
            description=metadata["description"]
        )
        tools.append(tool)
    
    # Create orchestrator agent with all capability tools
    orchestrator = Agent(
        name="AI Assistant",
        instructions="""You are an intelligent assistant that can help with various tasks. 
        You have access to specialized agents through tools. Use the appropriate tool 
        based on what the user needs.""",
        model="o3",
        tools=tools
    )
    
    # Run the orchestrator agent with user prompt
    user_input = "Find our company policy on remote work"
    await Runner.run(orchestrator, user_input)

if __name__ == "__main__":
    asyncio.run(main())
```

## Why This Approach Works

**For Developers:**
- Build and test individual capabilities independently
- Add new features without touching existing code
- Debug issues in isolation

**For Operations:**
- Deploy updates to specific capabilities only
- Scale resources based on actual usage patterns
- Monitor and troubleshoot individual components

**For Business:**
- Faster feature development with parallel teams
- Reduced risk of system-wide failures
- Easy integration of new AI models and providers

## Getting Started

1. **Identify your AI capabilities** - What different things does your system need to do?
2. **Create separate plugins** - Build one capability per plugin agent
3. **Use the discovery pattern** - Let your system automatically find capabilities
4. **Start simple** - Begin with basic implementations and evolve

## The Bottom Line

Plugin architecture isn't just about code organization - it's about building AI solutions that can evolve with your business needs. Instead of rewriting everything when requirements change, you simply add, remove, or update individual capabilities.

Start small, think modular, and watch your AI solution become more maintainable, scalable, and powerful over time.