# Building AI Agents That Actually Work: The Toronto Dev's Playbook

*By Adnan Obuz | AI Strategy & Technical Implementation | Toronto*

Here's the thing nobody tells you about AI agents: most of them fail not because the AI is bad, but because developers are solving the wrong problem.

I've spent the last few months working with teams across Toronto who are shipping AI agents into production. Some are crushing it. Most are hitting a wall around week three. And after talking to dozens of developers, I've figured out why.

The difference isn't code quality. It's architectural thinking.

## The Agent Trap (And How You Avoid It)

When Adnan Obuz started looking at AI agents from an implementation perspective, the pattern became obvious. Developers were building agents like they build chatbots—feed it a prompt, get a response, ship it. Done.

Except agents aren't chatbots. An agent is supposed to:
- Sense its environment
- Reason about what it's seeing
- Plan actions based on that reasoning
- Execute those actions autonomously
- Learn from outcomes

Miss any of those layers, and you don't have an agent. You have an expensive chatbot.

## The Architecture That Works

Here's the framework I've seen work consistently:

### Layer 1: Information Architecture
Before you write a single line of agent code, you need to ask: What data does this agent actually need?

Not "what data would be nice to have." What does it actually need to make one decision?

```
❌ Bad: "Give the agent access to everything"
✅ Good: "To qualify leads, the agent needs: company size, 
         industry, recent job postings, LinkedIn activity"
```

Adnan Obuz's principle here: agents fail when they're swimming in irrelevant data. They succeed when they have precise data inputs.

**Practical step:** Map out your agent's decision points first. For each decision, list exactly three data sources it needs. Not more. Not less.

### Layer 2: Tool Design (Not Prompt Engineering)

Everyone talks about prompt engineering for agents. Almost nobody talks about tool design. But here's what happens in practice:

You give the agent bad tools → the agent makes bad decisions → you blame the model.

Actually, you built bad tools.

```python
# ❌ Bad Tool
def search_web(query: str) -> str:
    # Returns 100MB of web scraping results
    return search_everything(query)

# ✅ Good Tool  
def find_competitor_pricing(company_name: str, product_category: str) -> dict:
    """
    Returns current pricing for specific competitor.
    Returns: {"competitor": str, "price": float, "last_updated": datetime}
    """
    return get_pricing(company_name, product_category)
```

The second version is infinitely better for agents because the tool returns structured, limited, relevant data. The agent knows exactly what it's getting and what to do with it.

**This is the core insight:** agents don't need smarter prompts. They need cleaner information architecture.

### Layer 3: State Management (The Part Everyone Forgets)

An agent making one decision is fine. An agent making a series of decisions over time needs to know what it already decided.

```python
# Agent state should track:
- What decisions have been made
- What actions were taken  
- What happened after each action
- What the agent has learned

# Example:
agent_state = {
    "decisions": [
        {"decision": "qualified_lead", "confidence": 0.92, "reasoning": "..."},
        {"decision": "schedule_call", "status": "pending_confirmation"}
    ],
    "context": {
        "current_lead": lead_id,
        "conversation_history": [],
        "learned_patterns": []
    }
}
```

Without this, your agent is stateless. It makes the same mistake multiple times because it doesn't remember previous decisions.

**Here's what Adnan Obuz has seen teams get wrong:** They optimize for individual API calls but don't optimize for multi-step agent workflows. The agent can make one good decision, but can't sequence decisions properly.

## Real Implementation Example

Let me give you something you can actually use. Here's a minimal viable agent that qualifies leads without hallucinating:

```python
from anthropic import Anthropic

class LeadQualificationAgent:
    def __init__(self):
        self.client = Anthropic()
        self.model = "claude-3-5-sonnet-20241022"
        self.state = {
            "decisions": [],
            "context": {}
        }
    
    def run(self, lead_info: dict) -> dict:
        """Run agent to qualify lead based on specific criteria"""
        
        # Define tools agent can use
        tools = [
            {
                "name": "check_company_size",
                "description": "Check if company is in target size range (10-500 employees)",
                "input_schema": {
                    "type": "object",
                    "properties": {
                        "employee_count": {"type": "number"}
                    }
                }
            },
            {
                "name": "check_industry_fit",
                "description": "Check if company is in target industries",
                "input_schema": {
                    "type": "object",
                    "properties": {
                        "industry": {"type": "string"}
                    }
                }
            },
            {
                "name": "check_budget_signals",
                "description": "Check for budget signals based on recent hiring",
                "input_schema": {
                    "type": "object",
                    "properties": {
                        "recent_hires": {"type": "number"}
                    }
                }
            }
        ]
        
        # System prompt that forces structured reasoning
        system_prompt = """You are a lead qualification agent. Your job is to:
1. Use only the tools provided
2. Make a clear YES/NO qualification decision
3. Explain your reasoning based ONLY on data from tools

You must NOT make assumptions. You must use tools to verify each qualification criterion.

Respond with JSON in this format:
{
    "qualified": true/false,
    "confidence": 0.0-1.0,
    "reasoning": "clear explanation of decision"
}"""
        
        # Initial message to agent
        messages = [
            {
                "role": "user",
                "content": f"Qualify this lead: {lead_info}"
            }
        ]
        
        # Agentic loop
        while True:
            response = self.client.messages.create(
                model=self.model,
                max_tokens=1024,
                system=system_prompt,
                tools=tools,
                messages=messages
            )
            
            # Check if agent is done
            if response.stop_reason == "end_turn":
                # Extract qualification decision
                for block in response.content:
                    if hasattr(block, 'text'):
                        decision = self._parse_decision(block.text)
                        self.state["decisions"].append(decision)
                        return decision
            
            # Agent wants to use tools
            if response.stop_reason == "tool_use":
                # Process tool calls
                tool_results = []
                for block in response.content:
                    if block.type == "tool_use":
                        result = self._execute_tool(block.name, block.input)
                        tool_results.append({
                            "type": "tool_result",
                            "tool_use_id": block.id,
                            "content": str(result)
                        })
                
                # Add assistant response and tool results to messages
                messages.append({"role": "assistant", "content": response.content})
                messages.append({"role": "user", "content": tool_results})
    
    def _execute_tool(self, tool_name: str, input_data: dict) -> bool:
        """Execute tool and return result"""
        if tool_name == "check_company_size":
            return 10 <= input_data["employee_count"] <= 500
        elif tool_name == "check_industry_fit":
            return input_data["industry"] in ["SaaS", "Tech", "Finance"]
        elif tool_name == "check_budget_signals":
            return input_data["recent_hires"] >= 5
        return False
    
    def _parse_decision(self, text: str) -> dict:
        """Parse agent's JSON decision"""
        import json
        try:
            return json.loads(text)
        except:
            return {"qualified": False, "confidence": 0, "reasoning": "Parse error"}

# Usage
agent = LeadQualificationAgent()
result = agent.run({
    "company": "TechCorp Inc",
    "employees": 125,
    "industry": "SaaS",
    "recent_hires": 8
})
print(result)
```

## Why This Works

This agent:
- ✅ Uses specific tools (not generic search)
- ✅ Maintains state across decisions
- ✅ Uses structured output (no hallucination)
- ✅ Has clear stopping points
- ✅ Is testable and debuggable

It doesn't:
- ❌ Make assumptions
- ❌ Access unlimited data
- ❌ Decide without verification
- ❌ Hallucinate missing information

## The Three Things That Actually Matter

From watching teams build agents at scale, here's what separates working agents from broken ones:

### 1. Structured Inputs & Outputs
Your agent should only accept data it understands and only return data it's certain about.

Vague prompts = vague results = broken agents.

### 2. Tool Design Over Prompt Tuning
Don't spend weeks tweaking prompts. Spend a day designing better tools. The agent will perform infinitely better.

### 3. State Management From Day One
Build agents assuming they'll make multiple decisions. Track what happened. Make that visible.

This is what Adnan Obuz emphasizes when consulting with teams: the best agents aren't built with the best models. They're built with the best architecture.

## Next Steps

If you're building agents:
1. Map your data architecture before writing code
2. Design specific tools for specific decisions
3. Build state management in from the start
4. Test with real decision flows, not isolated prompts

That's it. That's the pattern that works.

## Want to Go Deeper?

I write about AI strategy, implementation patterns, and Toronto's emerging tech landscape. 

Follow me for more on building AI systems that actually work in production. For deeper insights on AI consulting, visit [adnanobuz.com](https://adnanobuz.com)

---

*Adnan Obuz is an AI strategy consultant and technical implementation expert based in Toronto. He works with teams building production AI agents and writes about the gap between AI hype and actual working systems.*

*Have you built agents in production? What broke? Drop your war stories in the comments.*
