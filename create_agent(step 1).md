Bilkul pareshan mat hoiye, hum is notebook ke saare core concepts ko bilkul shuruat (ground zero) se samjhenge. Is notebook me **LangGraph** framework ka use karke ek dynamic AI Agent banana sikhaya gaya hai.

Is pure notebook ko hum **4 aasan Modules** me divide karke padhenge. Har module ka core concept aur uska ek chota, asardaar Python example niche diya gaya hai.

---

### **Module 1: ReAct (Reason + Act) Loop Kya Hai?**

#### **Core Concept:**
Normal LLM (jaise GPT ya Claude) bina soche-samjhe ek hi baar me answer generate karne ki koshish karte hain, jisse wo maths ya dynamic tasks me galat ho jaate hain. 
**ReAct** ek aesa tarika hai jahan LLM ek loop me kaam karta hai:
1. **Reason (Sochna):** LLM sochega, "User ne maths ka sawal pucha hai. Mujhe iske liye calculator tool chalana hoga."
2. **Act (Karna):** LLM calculator tool ko call karega.
3. **Observe (Dekhna):** Tool se jo answer aayega, LLM use dekhega aur sochega ki kya kaam poora ho gaya ya agla step lena hai.

#### **Simple Implementation Example:**
Is loop ko chalane ke liye LangGraph hume ek prebuilt component deta hai jise `create_react_agent` bolte hain.

```python
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent

# 1. Model define karein
model = ChatOpenAI(model="gpt-4o")

# 2. Agent banayein (Abhi bina tools ke simple agent hai)
agent = create_react_agent(model, tools=[])
```

---

### **Module 2: Defining and Using Tools (LLM ko calculator dena)**

#### **Core Concept:**
LLM ke paas dimaag hota hai par hath-pair nahi hote. **Tools** LLM ke hath-pair hain. Python me kisi bhi normal function ke upar `@tool` decorator lagakar hum use agent ka tool bana sakte hain. LLM is function ki **Docstring** (comment description) ko padh kar samajhta hai ki ise kab aur kyun chalana hai.

#### **Implementation Example:**
```python
from langchain_core.tools import tool

# Humne ek simple calculator function ko 'tool' bana diya
@tool
def multiply_numbers(a: float, b: float) -> float:
    """Do numbers ko multiply karne ke liye is tool ka use karein."""
    return a * b

# Ab is tool ko hum apne agent ko de denge
agent_with_tools = create_react_agent(model, tools=[multiply_numbers])

# Jab user poochega "What is 3.1 * 4.2?", agent is tool ko use karega
response = agent_with_tools.invoke({
    "messages": [{"role": "user", "content": "3.1 * 4.2 ka result kya hoga?"}]
})
```
*Yahan LLM khud multiplication nahi karega, balki hamare `multiply_numbers` tool ko call karke accurate result laayega.*

---

### **Module 3: State Management aur `add_messages` Reducer**

#### **Core Concept:**
Agent jab loop me chalega, toh use apni conversation history (chats, tool call details, aur tool outputs) yaad rakhni hoti hai. Is memory ko LangGraph me **State** kehte hain.
* **Reducer:** Jab bhi koi naya message system me aata hai, toh purane messages ke sath use kaise update kiya jaye, yeh **Reducer** decide karta hai.
* **`add_messages`**: Yeh ek standard reducer hai jo naye messages ko purani list ke aakhir me jodta (append) jata hai.

#### **Implementation Example:**
```python
from typing import Annotated, Sequence
from typing_extensions import TypedDict
from langchain_core.messages import BaseMessage
from langgraph.graph.message import add_messages

# Hum state define kar rahe hain jisme messages save honge
class SimpleAgentState(TypedDict):
    # 'add_messages' ensures ki naya message list me append ho jaye, replace na ho
    messages: Annotated[Sequence[BaseMessage], add_messages]
```

---


### Prerequisite for Module 4:
# InjectedState in LangGraph

`InjectedState` ka use tool ko graph ki state automatically dene ke liye hota hai, bina LLM ke state pass karwaye.

---

# Scenario

State:

```python
state = {
    "user_id": 101,
    "messages": [
        {"role": "user", "content": "Show my orders"}
    ]
}
```

Hum ek tool banana chahte hain jo user ke orders fetch kare.

---

# 1. WITHOUT InjectedState

## Tool

```python
from langchain_core.tools import tool

@tool
def fetch_orders(user_id: int):
    print(f"Fetching orders for user {user_id}")

    return [
        {"order_id": 1, "amount": 500},
        {"order_id": 2, "amount": 800}
    ]
```

## LLM Tool Call

```json
{
  "user_id": 101
}
```

## Flow

```text
User
  ↓
LLM
  ↓
LLM decides:
fetch_orders(user_id=101)
  ↓
Tool executes
```

### Problem

Agar state me `user_id=101` hai, to LLM ko ye value prompt se milni chahiye.

Agar LLM galti se:

```json
{
  "user_id": 999
}
```

bhej de, to galat user ka data fetch ho sakta hai.

---

# 2. WITH InjectedState

## Tool

```python
from typing import Annotated
from langchain_core.tools import tool
from langgraph.prebuilt import InjectedState

@tool
def fetch_orders(
    state: Annotated[dict, InjectedState]
):
    user_id = state["user_id"]

    print(f"Fetching orders for user {user_id}")

    return [
        {"order_id": 1, "amount": 500},
        {"order_id": 2, "amount": 800}
    ]
```

## LLM Tool Call

```json
{}
```

LLM ne `user_id` pass hi nahi kiya.

---

## LangGraph Internally

Tool execute hone se pehle:

```python
state = {
    "user_id": 101,
    "messages": [...]
}
```

inject kar diya jata hai.

Actual execution:

```python
fetch_orders(state)
```

Tool ke andar:

```python
user_id = state["user_id"]
```

mil jata hai.

---

# Visual Difference

## Without InjectedState

```text
State
  ↓
Prompt
  ↓
LLM
  ↓
Tool Call:
{
  "user_id": 101
}
  ↓
Tool
```

State → LLM → Tool

---

## With InjectedState

```text
State
  ├──────────────┐
  │              │
  ▼              │
LangGraph        │
  ▼              │
Tool Call {}     │
  ▼              │
Tool <───────────┘
```

State directly Tool tak pahunchti hai.

LLM ko state ka pata bhi nahi.

---

# More Realistic Example

User:

```text
Show my last 5 orders
```

## Without InjectedState

```python
@tool
def get_orders(user_id: int, limit: int):
    ...
```

LLM call:

```json
{
  "user_id": 101,
  "limit": 5
}
```

---

## With InjectedState

```python
@tool
def get_orders(
    limit: int,
    state: Annotated[dict, InjectedState]
):
    user_id = state["user_id"]

    return db.get_orders(
        user_id=user_id,
        limit=limit
    )
```

LLM call:

```json
{
  "limit": 5
}
```

LangGraph automatically:

```python
state["user_id"] = 101
```

inject karega.

---

# Security Benefit

State:

```python
state = {
    "user_id": 101,
    "jwt_token": "secret-token",
    "email": "abc@gmail.com"
}
```

Without `InjectedState`, sensitive information prompt me expose ho sakti hai.

With `InjectedState`, sensitive information LLM ko dikhaye bina tool use kar sakta hai.

---

# Key Benefits

- LLM ko unnecessary state pass nahi karni padti.
- Hallucinated arguments ka risk kam hota hai.
- Sensitive information hidden rehti hai.
- Prompt size aur token usage kam hota hai.
- Tools directly trusted runtime state use karte hain.

---

# One-Line Summary

`InjectedState` ka main purpose sirf state access dena nahi hai, balki runtime state ko safely, efficiently aur reliably tools tak pahunchana hai bina LLM par depend kiye.




### **Module 4: Tools ke andar se State ko Access aur Modify karna (Advanced)**

#### **Core Concept:**
Yeh is notebook ka sabse mahatvapurna aur anokha part hai. Maan lijiye aap chahte hain ki calculator tool jab bhi chale, toh woh ek custom state list (jaise ki `ops: List[str]`) ke andar calculations ka record (history) save karta jaye.

* **Problem:** LLM ko hum direct state pass karne ko nahi bol sakte kyunki use hamari internal state ke baare me nahi pata.
* **Solution (`InjectedState`):** Hum tool ke argument me `InjectedState` use karte hain. Isse LLM se woh state parameter chhup jata hai, par jab python code sach me chalta hai, toh LangGraph usme automatically current state daal deta hai.
* **State Update (`Command`):** Tool chalne ke baad, hum `Command(update={...})` return karte hain jo state ko update kar deta hai.

#### **Implementation Example:**
```python
from typing import List
from langgraph.prebuilt import InjectedState
from langgraph.types import Command
from langchain_core.tools import tool

# Custom state jisme hum 'ops' (operations list) maintain karenge
class CustomState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]
    ops: Annotated[List[str], lambda x, y: x + y] # Purani list me nayi list jodne ke liye simple reducer

@tool
def smart_calculator(a: float, b: float, state: Annotated[dict, InjectedState]) -> Command:
    """Math calculations karne aur history save karne ke liye."""
    result = a * b
    
    # Tool ne state se current 'ops' (operations list) ko read kiya
    current_ops = state.get("ops", [])
    log_msg = f"Multiplied {a} with {b} to get {result}"
    
    # Command ke jariye hum result bhi return kar rahe hain aur state ko update bhi kar rahe hain
    return Command(
        update={"ops": [log_msg]}, # State ki 'ops' list me naya log add ho jayega
        value=result               # Yeh output LLM ko milega
    )
```

---
```plaintext

# FINAL IMPLEMENTATION OF CORE CONCEPT:

###############################################################################################################################
import os
from typing import Annotated, List, Sequence
from typing_extensions import TypedDict
from langchain_core.messages import BaseMessage, HumanMessage, ToolMessage  # <-- Added ToolMessage
from langchain_core.tools import tool, InjectedToolCallId  # <-- Added InjectedToolCallId
from langchain_google_genai import ChatGoogleGenerativeAI
from langgraph.graph.message import add_messages
from langgraph.prebuilt import InjectedState, create_react_agent
from langgraph.types import Command
from langgraph.managed import RemainingSteps

# =====================================================================
# 1. Reducer Function (State Logs Update)
# =====================================================================
def reduce_ops(left: List[str] | None, right: List[str] | None) -> List[str]:
    if left is None:
        left = []
    if right is None:
        right = []
    return left + right

# =====================================================================
# 2. Custom State Definition
# =====================================================================
class CustomState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]
    ops: Annotated[List[str], add_messages]
    remaining_steps: RemainingSteps

# =====================================================================
# 3. Tool Definition with InjectedState, InjectedToolCallId and Command
# =====================================================================

@tool
def smart_calculator(
    a: float, 
    b: float, 
    tool_call_id: Annotated[str, InjectedToolCallId], # <-- Automatically gets the Tool Call ID from system
    state: Annotated[dict, InjectedState]
) -> Command:
    """Math calculations karne aur operation logs ko state me save karne ke liye is tool ka use karein.
    Ye tool 'a' aur 'b' ko multiply karega."""
    result = a * b
    
    # State se purani logs padhi
    current_ops = state.get("ops", [])
    log_msg = f"Calculated: {a} * {b} = {result}"
    
    # Tool output ko 'messages' ke andar 'ToolMessage' banakar bheja jata hai
    # Isme 'tool_call_id' pass karna zaroori hai
    return Command(
        update={
            "ops": [log_msg],  # custom state update
            "messages": [
                ToolMessage(
                    content=f"Multiplication result is {result}.", 
                    tool_call_id=tool_call_id
                )
            ]  # tool message output update
        }
    )

# =====================================================================
# 4. Agent Initialization & Interactive Loop
# =====================================================================
if __name__ == "__main__":
    
    # TODO: Apni Gemini API key yahan paste karein
    GEMINI_API_KEY = "AQ.Ab8RN6JRKXR33eZqhpbLRZCQiUJzgLwJotxHn5hKHA"  
    
    if GEMINI_API_KEY == "YOUR_GEMINI_API_KEY_HERE":
        print("Error: Please put your real Gemini API key in the code first.")
        exit()

    print("Initializing Gemini Model...")
    model = ChatGoogleGenerativeAI(
        model="gemini-2.5-flash",
        api_key=GEMINI_API_KEY,
        temperature=0
    )
    
    print("Creating React Agent...")
    agent = create_react_agent(
        model,
        tools=[smart_calculator],
        state_schema=CustomState
    )
    
    # Shuruat me empty state banayi
    current_state = {
        "messages": [],
        "ops": []
    }
    
    print("\n" + "="*50)
    print("Welcome to Agent Chat! (Type 'exit' to quit the session)")
    print("="*50)
    
    # Interactive Loop
    while True:
        try:
            # User se console me message lena
            user_input = input("\nYou: ")
            
            # Exit check
            if user_input.strip().lower() == "exit":
                print("\nEnding chat session. Goodbye!")
                break
            
            if not user_input.strip():
                continue
            
            # User ke naye message ko current state ki messages list me append kiya
            current_state["messages"].append(HumanMessage(content=user_input))
            
            # Agent ko invoke kiya purani state ke sath
            current_state = agent.invoke(current_state)
            
            # Agent ka aakhiri generated response print kiya
            last_message = current_state["messages"][-1]
            print(f"Agent: {last_message.content}")
            
            # State ke operation logs print karna
            current_logs = current_state.get('ops', [])
            if current_logs:
                print(f"[Ops System Logs: {current_logs}]")
            print(current_state["messages"] or []
            )
        except Exception as e:
            print(f"\nAn error occurred: {e}")
            break
```

### **Summary of the Flow (Notebook me kya ho raha hai?):**
1. Sabse pehle API setup hota hai.
2. Ek custom **State** banti hai (jo messages aur operations history ko track karti hai).
3. Ek **Smart Tool** banta hai jo chalne ke sath-sath graph ki state ko update karta hai (`InjectedState` aur `Command` ka use karke).
4. `create_react_agent` ko model, tools, aur custom state dekar agent ready kiya jata hai.
5. Agent ko invoke karke testing ki jati hai aur console me print karke dekha jata hai ki kaise state update ho rahi hai aur conversation manage ho rahi hai.
