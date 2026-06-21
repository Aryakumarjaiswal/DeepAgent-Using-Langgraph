---

### **Part 1: Abhi tak aapne kya sikha? (5 Key Points)**

1. **ReAct (Reasoning + Acting) Architecture:** Aapne seekha ki ek modern AI agent direct answer dene ke bajaye ek step-by-step loop me chalta hai (Sochta hai -> Action/Tool call karta hai -> Output observe karke agla decision leta hai).
2. **Dynamic Tools via `@tool`:** Aapne seekha ki kaise ordinary Python functions ke upar `@tool` lagakar unhe ek structured JSON schema me badla jata hai, jise LLM padhkar sahi waqt par call kar sake.
3. **Graph State Management:** LangGraph me memory ko manage karne ke liye **State** (`TypedDict`) ka use hota hai, jahan conversations aur custom logs (jaise aapki `ops` list) pure loop ke dauran save rehte hain.
4. **State Reducers (`add_messages` aur customized reducers):** Purane data ko overwrite hone se bachane ke liye reducers ka roll samajh aaya. Jaise `add_messages` chats ko end me append karta hai, aur kaise humne custom list reducer banakar operations list (`ops`) ko safe tareeqe se merge kiya.
5. **State Injection & Control (`InjectedState` & `Command`):** Sabse advanced concept! Kaise hum tool parameters ko LLM se chhupate hain (`InjectedState`) taaki graph khud real-time state tool me pass kar sake. Aur kaise tool `Command` return karke system memory ko modify karta hai.

---

### **Part 2: Notebook `1_todo.ipynb` (Task Planning Foundations) ka Core Crux**

Jaise-jaise aap complex tasks (jaise Deep Research ya Reports) build karenge, waise-waise traditional simple agents fail hone lagte hain. Is notebook ka yahi main core focus hai.

#### **1. Main Problem jise ye solve karta hai: Task Drift (Bhatak Jana)**
Agar aap kisi simple agent ko bolein: *"Internet par 5 compnies ke financial trends search karo, unhe compare karo, aur ek pdf report banao."*
* **Problem:** Agent 2-3 searches ke baad bhatak jayega (task drift). Use yaad nahi rahega ki usne kya-kya dhoondh liya hai aur aage kya dhoondhna bacha hai. Uske token limits khatam hone lagengi aur woh loop me phans jayega.

#### **2. Core Concept: Structured Planning (TODO List)**
Aapne real life me dekha hoga ki bade projects ko solve karne ke liye hum ek checklist (TODO list) banate hain. `1_todo.ipynb` agent ke sath bhi bilkul yahi karta hai. 

Isme agent ki State me ek **`todos`** list add kar di jati hai:
```python
class Todo(TypedDict):
    task: str
    status: Literal["pending", "in_progress", "completed"]
```

#### **3. Notebook ke 2 important Tools (Harness):**
Agent ke paas dimaag hota hai par use plan update karne ke liye **planning tools** diye jaate hain:
* **`write_todos(todo_list)`:** Jab user koi complex query puchega, agent sabse pehle koi action nahi lega. Woh is tool ko call karke pure task ko chote-chote sub-tasks me todkar ek systematically ordered TODO list state me save karega.
* **`read_todos()`:** Har naye step par, agent is tool ke zaroor checklist ko check karega ki: 
  * *"Kaunsa task complete ho gaya?"*
  * *"Abhi kis par kaam chal raha hai (in_progress)?"*
  * *"Agla pending task kya hai?"*

#### **4. Dynamic Re-planning (Plan ko adjust karna)**
Agar complex research ke dauran agent ko beech me koi naya unexpected data milta hai, toh woh `write_todos` tool ka use karke beech me hi apni list ko overwrite ya update kar sakta hai. Yeh use kafi "strategic" aur flexible banata hai.

---

### **Aapke Module 3 ke liye Ek Chota mental model:**
Aap is module me seekhenge ki ek **"Shallow Agent"** (bina plan ka bhatakta hua agent) aur ek **"Deep Agent"** (jo pehle plan banata hai, checklist track karta hai aur focused rehta hai) ke beech me kya antar hota hai. 

Agle notebook me aap dekhenge ki kaise agent user ki query milte hi sabse pehle `write_todos` tool se apni diary (state) me checklist likhta hai.
