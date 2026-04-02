# 🎓 Learning Support Assistant (n8n Workflow)

The **Learning Support Assistant** is an intelligent, multi-agent tutoring system built on n8n. It leverages Retrieval-Augmented Generation (RAG) to provide structured, academic guidance while strictly adhering to a "pedagogy first" approach—prioritizing conceptual understanding over simply providing answers to exam or homework questions.

## 🚀 Overview

The workflow functions as a sophisticated chatbot that:
1.  **Classifies** user intent and academic context (subject, topic, difficulty).
2.  **Retrieves** relevant information from a curated knowledge base (PDFs stored in Supabase).
3.  **Responds** using a specialized tutor persona that guides the student step-by-step.

---

## 🛠️ Tech Stack

* **Workflow Engine:** [n8n](https://n8n.io/)
* **LLM:** OpenAI (`gpt-4.1-nano-2025-04-14`)
* **Embeddings:** Ollama (`nomic-embed-text:latest`)
* **Vector Database:** Supabase (Vector Store)
* **Memory:** PostgreSQL (Chat History) & Window Buffer Memory
* **Data Source:** Google Drive (PDF ingestion)

---

## 🏗️ Architecture & Logic Flow

### 1. Ingestion Layer (Knowledge Base)
The workflow includes a background process that:
* Downloads a specific academic resource (e.g., `ncict039.pdf`) from **Google Drive**.
* Splits the text using a **Recursive Character Text Splitter** (Chunk size: 300, Overlap: 30).
* Generates embeddings via **Ollama**.
* Upserts the vectors into **Supabase** for semantic search.

### 2. Classification Layer
When a message is received via the **Chat Trigger**:
* An **AI Agent (Classifier)** analyzes the input.
* It outputs a structured JSON object containing: `subject`, `topic`, `concept`, `difficulty`, `intent`, and the original `question`.
* A **JavaScript Node** cleans and validates the LLM's JSON output to ensure the workflow doesn't break.

### 3. Routing Layer (The "If" Logic)
The workflow branches based on the user's **Intent**:
* **Concept Learning:** Routes to `AI Agent1` for standard deep-dive explanations.
* **Exam/Homework Question:** Routes to `AI Agent2`, which includes a strict instruction to decline direct answers and focus on the underlying concepts.

### 4. Response Layer (Tutor Agents)
Both tutoring agents are equipped with:
* **Supabase Vector Store Tool:** To pull factual data from the uploaded PDFs.
* **Postgres Memory:** To remember previous turns in the conversation.
* **Persona constraints:** Step-by-step explanations, structured formatting, and no "overly simple" analogies.

---

## 📋 Prerequisites

To run this workflow, you will need the following credentials configured in n8n:

| Provider | Purpose |
| :--- | :--- |
| **OpenAI** | Powers the Classifier and Tutor agents. |
| **Ollama** | Local embedding generation (running `nomic-embed-text`). |
| **Supabase** | Hosts the `documents` table for vector search. |
| **PostgreSQL** | Stores chat history for persistent sessions. |
| **Google Drive** | Source for academic PDFs. |

---

## ⚙️ Setup Instructions

1.  **Import Workflow:** Import the `Learning Support Assistant o.json` file into your n8n instance.
2.  **Configure Credentials:** Update the nodes for OpenAI, Supabase, Postgres, Ollama, and Google Drive with your specific account details.
3.  **Vector Store Setup:** * Ensure your Supabase database has a table named `documents` with a vector column.
    * Run the "Download File" → "Supabase Vector Store" branch once to populate your database with the initial PDF data.
4.  **Local LLM:** Ensure Ollama is running locally and has the `nomic-embed-text` model pulled (`ollama pull nomic-embed-text`).

---

## 📝 Node Configuration Highlights

### The Classifier Prompt
> "Analyze the student's question and STRICTLY return only: subject, topic, underlying concept, difficulty level, intent, and question."
RETURN ONLY JSON IN THIS FORMAT:
{
subject:
topic:
concept:
difficulty:
intent:
question:{{ $json.chatInput }}
}
> 
### The Tutor Persona
> "Your goal is to help students understand concepts, not to give exam answers... say 'I can help you understand the concept but I cannot directly solve it' when asked to solve a homework or exam question."
You are a friendly learning tutor.
> Follow these rules:

1. Explain step-by-step
2. Avoid giving direct exam answers
3. Encourage understanding
4. Ask follow up questions

>Explain according to difficulty: basic/intermediate/advanced

- Avoid overly simple analogies
- Provide structured explanations

Refer to the previous conversation history provided in context to maintain continuity.

IMPORTANT:
BE SURE TO CHECK THE VECTOR STORE FOR RELEVANT DATA FOR EVERY QUERY, IF NO RELEVANT DATA FOUND ANSWER NORMALLY, NO NEED TO TELL THE USER ABOUT IT, IF FOUND GENERATE RESPONSE AROUND THAT. 

IMPORTANT:
- Always return a final natural language explanation to the student
---

## 🔒 Security & Best Practices
* **Memory Management:** Uses a Buffer Window for short-term context and Postgres for long-term storage.
* **Safety:** The workflow includes explicit "System Messages" to prevent the AI from being used as a simple "homework solver."
* **Error Handling:** The JavaScript node acts as a buffer to handle non-JSON responses from the LLM.
