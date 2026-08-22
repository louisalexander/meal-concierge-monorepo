# Meal Concierge

**An AI concierge for what you eat.**

Monorepo for the Meal Concierge microservices — a conversational meal assistant that plans,
substitutes, scales and explains, and answers *from documents you control* rather than from
whatever a language model happens to remember.

📖 **[Read the full overview →](https://louisalexander.github.io/meal-concierge-monorepo/)**

---

## About

Food questions look casual and are anything but. *Can I swap buttermilk for yogurt here?*
*Is this safe with a tree-nut allergy?* *What do I cook with what's left in the fridge?*
Each one depends on a specific recipe, a specific pantry, a specific person — and getting it
subtly wrong ranges from a ruined dinner to a genuine medical problem.

A general-purpose chatbot answers those questions fluently and, often enough, incorrectly. It has
no idea which recipe you are holding, and it cannot cite where an answer came from.

Meal Concierge takes the opposite approach. Every answer is retrieved from a corpus you curate —
your recipes, your nutrition references, your dietary guidance — embedded into a vector store and
handed to the model as context at question time. The model does the reasoning and the phrasing;
your documents supply the facts.

Three things follow from that:

- **Grounded, not guessed.** Retrieval-augmented generation over a vector store you own. Swap the
  corpus, change the expertise.
- **It remembers the thread.** Per-conversation memory, so "halve that" and "make it dairy-free"
  mean what you'd expect.
- **Services, not a monolith.** One repo, independently deployable Spring Boot services, and a
  shared Maven archetype for generating the next one.

## How it works

1. **Ingest** — Apache Tika reads the source document at application startup.
2. **Chunk** — a token-aware splitter breaks the text into passages small enough to retrieve
   precisely and large enough to stay coherent.
3. **Embed & store** — each chunk is embedded via OpenAI and written to a Qdrant collection, which
   the service provisions on first run.
4. **Retrieve** — an incoming question becomes a vector query; the nearest passages come back as
   evidence.
5. **Answer** — retrieved passages plus the conversation history go to the chat model, which
   returns a structured, typed response.

> **Repository status.** The pipeline above is real and running in `chat-service`; the
> meal-specific corpus and the services around it are the work in progress. The service ships with
> a sample document so you can clone it and get a grounded answer in one command.

## Services

| Service | Status | Description |
| --- | --- | --- |
| [`chat-service`](./chat-service) | Live | The conversational core. Loads and indexes the corpus on startup, then serves `POST /ask` with retrieval and per-conversation memory attached. |

```bash
curl -X POST http://localhost:8080/ask \
  -H 'Content-Type: application/json' \
  -H 'X_CONV_ID: dinner-tonight' \
  -d '{"question": "What can I substitute for buttermilk?"}'

# {"answer": "..."}
```

Pass an `X_CONV_ID` header to keep separate threads apart; omit it and everything lands in one
default conversation.

The repository root is itself a **Maven archetype** — the template each new Meal Concierge service
is generated from, so the next service starts with the same layout, conventions and build wiring
as this one.

## Stack

| Layer | Choice | Why |
| --- | --- | --- |
| Runtime | Java 17 · Spring Boot 3.4 | Long-term-support baseline with first-class observability and config. |
| AI orchestration | Spring AI | Advisor pipeline composes retrieval, memory and logging without hand-rolled glue. |
| Vector store | Qdrant | Fast similarity search, runs locally in Docker, schema created on boot. |
| Models | OpenAI | Embeddings and chat behind one provider today; the abstraction allows others. |
| Ingestion | Apache Tika | One reader for PDFs, documents and the messy formats recipes actually arrive in. |
| Local infra | Docker Compose | Spring Boot starts and stops Qdrant with the app — no separate setup step. |
| Build | Maven multi-module | One reactor build; the root doubles as the archetype for new services. |

## Getting started

Docker and a JDK 17+ are the only prerequisites — Compose brings up Qdrant for you.

```bash
git clone https://github.com/louisalexander/meal-concierge-monorepo.git
cd meal-concierge-monorepo/chat-service

export OPENAI_API_KEY=sk-...
./mvnw spring-boot:run
```

On startup the service brings up Qdrant, indexes the bundled document, and begins serving
`POST /ask` on port 8080. Point `app.resource` in `application.properties` at your own file to
change what it knows.

## Where this is going

- [x] Document ingestion, chunking and embedding at startup
- [x] Qdrant-backed retrieval wired into the chat pipeline
- [x] Per-conversation memory and typed responses over `POST /ask`
- [x] Maven archetype at the root for generating the next service
- [ ] A real meal corpus — recipes, nutrition data, allergen references
- [ ] Citations returned alongside each answer
- [ ] Pantry and preference service, so answers know your kitchen
- [ ] Persistent conversation memory beyond a single process
- [ ] Meal planning and shopping-list generation across a week
