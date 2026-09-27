```
# MASTER STUDY GUIDE: **Multi‑Agent Systems (MAS) in LangGraph**
```

```
## **1. THE BIG PICTURE & CORE THEORY**
```

# `### **The Core Problem**` 

```
Modern AI systems often need to perform **multiple types of reasoning or
tasks**, each requiring different skills. A single LLM acting alone becomes
rigid, overloaded, or inefficient.
**Multi-Agent Systems (MAS)** solve this by allowing **multiple specialized
agents** to collaborate, each performing a specific role.
```

```
> “Multi Agentic Flow means more than one agent… more than one agent.”
```

# `### **Foundational Concepts**` 

# `#### **Node**` 

```
A *node* is simply a **function**.
In LangGraph, every agent or processing step is represented as a node.
```

# `#### **Edge**` 

```
An *edge* is the **connection** between nodes.
It defines **who runs next**.
```

```
Types of edges:
```

```
- **Normal Edge:** Fixed sequence (A → B → C)
- **Conditional Edge:** Dynamic decision (A → B or A → C depending on state)
```

```
#### **Deterministic Flow**
```

```
A workflow where **all steps are predefined**.
```

```
#### **Dynamic Flow**
```

```
A workflow where **decisions are made at runtime**, often by the LLM.
```

```
#### **Agentic Flow**
```

```
A flow where the **LLM decides which tool or node to call next**.
```

```
#### **Partial Agentic Flow**
```

```
A deterministic flow with **limited autonomy**.
```

# `### **Real‑World Analogy**` 

```
Think of a **hospital emergency room**:
```

- `A **triage nurse** (router) decides whether a patient goes to cardiology, orthopedics, or general medicine.` 

- `Each department has **specialists**.` 

- `Specialists may perform multiple steps (tests, scans, diagnosis).` 

```
LangGraph MAS works the same way:
```

```
- Router → Specialist → Tools → Final Answer.
```

```
## **2. UNDER THE HOOD: MECHANICS & ARCHITECTURE**
```

```
### **How It Works (Step-by-Step)**
```

`1. **User Query Arrives**` 

`2. **Router Node Reads the Query**` 

`3. Router decides:` 

- `Technical` 

- `Billing` 

- `General` 

`4. Router sends the query to exactly **one specialist**.` 

```
5. Specialist produces the final answer.
```

`6. Workflow ends.` 

```
### **Mental Model: MAS Architecture**
```

```
| Component | Role | Analogy |
|----------|------|---------|
| Router | Chooses the right specialist | Triage nurse |
| Specialist Agent | Solves the problem | Doctor |
| Tools | External capabilities | Medical tests |
| Edges | Workflow transitions | Hallways between rooms |
| State | Shared memory | Patient chart |
```

```
### **Instructor’s Structural Diagram**
```
Agent 1:
    LLM
    Tool1, Tool2, Tool3, Tool4
```

```
Agent 2:
    LLM
    Tool1, Tool2, Tool3, Tool4
```

```
Agent 3:
    LLM
    Tool1, Tool2, Tool3, Tool4
```
```

```
Agents communicate → Multi-agentic system.
```

```
### **Pattern Comparison (From Notebook)**
```

```
| # | Pattern | Control Style | Typical Calls |
|---|---------|---------------|---------------|
| 1 | Router | Select one specialist | 2 |
| 2 | Subagents | Agent delegates through tools | Variable |
```

- `| 3 | Supervisor | Supervisor selects workers | Variable |` 

```
| 4 | Handoff | Triage transfers control | 2 |
```

```
| 5 | Swarm | Agents hand control to each other | Variable |
```

- `| 6 | Hierarchical Supervisor | Supervisor selects team subgraph | 3 |` 

- `| 7 | Parallel Collaboration | Dynamic fan-out | N + 2 | | 8 | Coordinator Peer Network | Coordinator selects peer | Variable |` 

```
## **3. STEP-BY-STEP HANDS-ON IMPLEMENTATION**
```

```
### **The Workflow**
We implement **Pattern 1: Router Multi-Agent**, exactly as demonstrated in the
notebook.
```

```
The flow:
```
```

```
User → Router → Specialist → Response
```
```

```
### **Annotated Code Blueprint**
```

```
### **Setup**
```python
```

```
from typing import TypedDict, Literal, Annotated
from pydantic import BaseModel
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, START, END
from langgraph.types import Command
# Load model
model = ChatOpenAI(
    model="gpt-4.1-mini",   # Small, fast model for routing
    temperature=0           # Deterministic output for routing decisions
)
```
---
### **Router Decision Schema**
```python
class RouterDecision(BaseModel):
    route: Literal["technical", "billing", "general"]
    # The router must choose exactly one specialist
```
```python
router_model = model.with_structured_output(RouterDecision)
# Ensures the model returns a structured JSON with a 'route' field
```
---
### **State Definition**
```python
class RouterState(TypedDict):
    query: str      # User question
    route: str      # Selected specialist
    response: str   # Final answer from specialist
```
---
### **Router Node**
```python
def router_node(state: RouterState):
    # Ask the model to classify the query into one specialist
    decision = router_model.invoke(
        f"""
Route the following user query to exactly one specialist.
Available specialists:
- technical
- billing
- general
User query:
{state["query"]}
"""
    )
    return {"route": decision.route}   # Store the chosen route in state
```
---
### **Specialist Nodes**
```python
def technical_agent(state: RouterState):
```

```
    # Technical specialist answers technical issues
    response = model.invoke(
        f"""
You are a technical support specialist.
Answer this query:
{state["query"]}
"""
    )
    return {"response": response.content}
```
```python
def billing_agent(state: RouterState):
    # Billing specialist answers payment issues
    response = model.invoke(
        f"""
You are a billing support specialist.
Answer this query:
{state["query"]}
"""
    )
    return {"response": response.content}
```
```python
def general_agent(state: RouterState):
    # General specialist answers general questions
    response = model.invoke(
        f"""
You are a general support specialist.
Answer this query:
{state["query"]}
"""
    )
    return {"response": response.content}
```
---
### **Routing Logic**
```python
def route_to_specialist(state: RouterState):
    return state["route"]   # Router decides next node
```
---
### **Build the Graph**
```python
def build_router_multi_agent():
    builder = StateGraph(RouterState)
    builder.add_node("router", router_node)
    builder.add_node("technical", technical_agent)
    builder.add_node("billing", billing_agent)
    builder.add_node("general", general_agent)
    builder.add_edge(START, "router")   # Start → Router
    builder.add_conditional_edges(
        "router",
        route_to_specialist,            # Router decides next node
        {
            "technical": "technical",
```

```
            "billing": "billing",
            "general": "general",
        },
    )
    builder.add_edge("technical", END)
    builder.add_edge("billing", END)
    builder.add_edge("general", END)
    return builder.compile()
```
```

```
### **Test Case**
```python
def run_router_multi_agent():
    graph = build_router_multi_agent()
    result = graph.invoke({
        "query": "My subscription payment was charged twice.",
        "route": "",
        "response": "",
    })
```

```
    print("Selected Agent:", result["route"])
    print("Response:", result["response"])
```
```

```
Expected:
- Route → **billing**
- Billing specialist answers.
```

```
## **4. QUICK REVISION CHEATSHEET & EXAM PREP**
```

```
### **Common Gotchas & Edge Cases**
```

- `Forgetting to return a dictionary from nodes → graph breaks.` 

- `Missing `END` edges → infinite loops.` 

- `Router must return **exact route strings** matching node names.` 

- `State keys must match the `TypedDict` exactly.` 

```
### **Top 3 Core Takeaways**
```

- `MAS = multiple specialized agents collaborating through nodes and edges. - Router pattern = simplest MAS architecture; one agent chooses the next. - LangGraph enables autonomy through conditional edges and structured state.` 

