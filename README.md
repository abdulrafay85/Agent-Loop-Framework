# Agent Loop - Lightweight Agent Flow Framework

This is a minimalistic agent framework developed for **personal practice and experimentation**. It demonstrates the core logic of how multi-agent handoff and dialogue processing can be orchestrated using simple Python classes.

---

## Purpose

**I made this system only for my practice. It has a basic agent coordination structure where handoffs are managed. Right now:**

* Agent handoff logic is there
* Memory support is not added yet (we'll add it later)
* Tool calling feature is also not added yet

**In future versions:**

* Agents will be able to maintain context (via memory)
* Tools will be added to fetch realtime data

**The main goal was to understand the basic flow of agentic systems. This code is made simple.**

**I made all this for my own practice so I can get a clear understanding of agentic architecture**

---

### Agent

Represents a single autonomous agent with optional handoffs to other agents.

```python
class Agent:
    def __init__(self, name, instructions, model, handoffs=None, handoff_description=None):
        self.name = name
        self.instructions = instructions
        self.model = model
        self.handoffs = {a.name: a for a in (handoffs or [])}
        self.handoff_description = handoff_description
```

### AgentLoop

Manages the flow of conversation between agents, handling user queries and agent responses.

```python
class AgentLoop:
    def __init__(self, agents: dict[str, Agent], start_key: str, query: str, max_turns=5):
        self.agents = agents
        self.current_key = start_key
        self.query = query
        self.max_turns = max_turns

    def run(self):
      turn = 0
      current_input = self.query

      while True:
        turn += 1
        print(f"Step {turn}")
        if turn > self.max_turns:
            raise Exception("MaxTurnsExceeded")

        agent = self.agents[self.current_key]

        # Build handoff info
        handoff_info = []
        for name, ag in agent.handoffs.items():
            tool_name = f"transfer_to_{name}"
            desc = ag.handoff_description or f"Use this to handoff to {name}."
            handoff_info.append(f"{tool_name}: {desc}")
        handoff_text = "Available handoffs:\n" + "\n".join(handoff_info)

        # Prepare messages
        messages = [
            {"role": "system", "content": agent.name},
            {"role": "system", "content": agent.instructions},
            {"role": "system", "content": handoff_text},
            {"role": "user",   "content": current_input},
        ]

        ai_msg = agent.model.invoke(messages)
        text = ai_msg.content.strip()

        if text.lower().startswith("handoff:"):
            _, target = text.split(":", 1)
            target = target.strip()
            if target in agent.handoffs:
                print(f"[handoff] {agent.name} → {target}")
                self.current_key = target
                continue
            else:
                return f"Error: no agent named '{target}' to handoff to."

        return text
```

---

## Example Setup

```python
# Initialize agents
history_tutor = Agent(name="History Tutor", instructions="You answer historical questions.", model=model)
cooking_tutor = Agent(name="Cooking Tutor", instructions="You help with cooking.", model=model)

# Coordinator agent
triage = Agent(
    name="Triage Agent",
    instructions="Route to History or Cooking Tutor.",
    model=model,
    handoffs=[history_tutor, cooking_tutor]
)

agents = {
    "Triage Agent": triage,            
    "History Tutor": history_tutor,    
    "Cooking Tutor": cooking_tutor     
}

# Run
runner = AgentLoop(agents, start_key="Triage Agent", query="How to boil eggs?")
print(runner.run())
```

###  

### Explanation

* `start_key="Triage Agent"` means the query will first go to the **Triage Agent**.
* `agents = {...}` is a dictionary where each agent's name is the key, and the value is its **Agent object**.
* The **Triage Agent** has the authority to perform handoffs — it decides which query should go to which agent.
* If the response contains `handoff:Cooking Tutor`, then control is passed to the **Cooking Tutor**.
