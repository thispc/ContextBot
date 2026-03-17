🤖 ContextBot: Agentic Multimodal AI for Instagram Content Analysis
===================================================================

1\. Executive Summary
---------------------

**ContextBot** is an event-driven, multimodal AI agent residing natively within Instagram Direct Messages. It solves the friction of passive video consumption by allowing users to seamlessly query complex social media content (Reels, Carousels, static images) in real-time.

By leveraging an agentic LLM architecture, the system asynchronously extracts media via proxy scrapers, processes visual frames and audio transcripts, and maintains conversational context—all without requiring the user to leave the Instagram app. This project demonstrates proficiency in high-concurrency webhook handling, background task orchestration, and multimodal agent reasoning.

2\. Core User Journey (Event Flow)
----------------------------------

The application relies entirely on the Meta Messenger API for a zero-friction UI.

1.  **Trigger:** User encounters a complex Reel (e.g., a fast-paced coding tutorial).
    
2.  **Action:** User taps the native Instagram "Share" icon and sends the Reel via DM to @ContextBot.
    
3.  **Optimistic Acknowledgment (T+1s):** Bot instantly replies: _"👀 Link received. Downloading media and transcribing audio..."_
    
4.  **Processing (T+5s to T+15s):** Background workers extract the .mp4, sample visual frames, and run Speech-to-Text.
    
5.  **Ready State:** Bot replies: _"✅ Done. I've watched the video. What specific details do you need?"_
    
6.  **Agentic Chat:** User asks: _"What was the exact terminal command used at 0:15?"_ The bot references the multimodal context window and answers instantly.
    

3\. System Architecture & Tech Stack
------------------------------------

This architecture isolates the web server from the heavy ML inference to ensure high availability and compliance with third-party webhook constraints.

**ComponentTechnologyPurposeIngestion / API GatewayFastAPI (Python)**High-performance, async-native web framework to receive Meta webhooks.**Task Queue / BrokerCelery + Redis**Offloads heavy scraping and AI inference to background workers to prevent Meta API timeouts.**Media ExtractionApify (Instagram Scraper)**Proxied residential IP scraping to bypass Instagram's strict authentication walls and retrieve raw .mp4 URLs.**Audio ProcessingOpenAI Whisper (API)**Fast, timestamp-aware Speech-to-Text extraction.**Vision / Reasoning EngineGoogle Gemini 1.5 Pro**Massive context window (up to 2M tokens) natively processes video frames, audio transcripts, and user prompts simultaneously.**Database & CachingPostgreSQL + pgvector**Stores parsed "Context Documents" for viral videos. If a second user shares the same Reel, compute costs drop to $0 via vector retrieval.**Local TunnelingNgrok**Exposes the local development environment to Meta's servers for live webhook testing.

4\. System Requirements & Engineering Constraints
-------------------------------------------------

### Functional Requirements

*   **Webhook Verification:** Must implement Meta's hub.challenge GET request handshake for initial app verification.
    
*   **Signature Validation:** Must validate the X-Hub-Signature-256 header on incoming POST requests using the App Secret to ensure payloads actually come from Meta.
    
*   **Agentic Routing:** The core LLM must possess tool-calling capabilities to dynamically decide whether to trigger visual analysis, text summarization, or external web search.
    

### Non-Functional Requirements (The Technical Hurdles)

*   **The 20-Second Meta Timeout:** Meta requires a 200 OK response to all webhooks within 20 seconds. Multimodal video processing takes longer. The FastAPI server must immediately acknowledge the payload and push the payload ID to Redis.
    
*   **Rate Limiting:** Must respect OpenAI/Gemini API limits and Instagram scraping limits by implementing exponential backoff retries in Celery workers.
    
*   **Idempotency:** Webhooks can sometimes be delivered twice by Meta. The system must check the message\_id against the PostgreSQL database before kicking off an expensive AI job.
    

5\. Agentic AI Workflow (The Orchestration Layer)
-------------------------------------------------

The system does not use a rigid pipeline. It utilizes an **Agent Executor** pattern (implementable via LangGraph or raw Python routing).

*   **The Router Agent:** Evaluates the incoming webhook. Is it a new video, or a chat reply?
    
*   **The Tools:**
    
    1.  fetch\_instagram\_data(url): Triggers Apify to get media files.
        
    2.  extract\_multimodal\_context(media\_url): Sends the video to Gemini to generate the master "Context Document".
        
    3.  query\_vector\_cache(ig\_post\_id): Checks pgvector to see if this video has already been processed today.
        
*   **The Synthesis Loop:** Once the Context Document is built, it is injected into the System Prompt. The Agent enters a conversational loop, retaining the chat history using a sliding window memory (retaining the last 5 messages).
    

6\. Database Schema (PostgreSQL)
--------------------------------

To optimize for cost and speed, caching is mandatory.

**Table: users**

*   id (UUID, Primary Key)
    
*   ig\_user\_id (String, Unique) - Meta's app-scoped user ID.
    
*   created\_at (Timestamp)
    

**Table: posts\_cache**

*   ig\_post\_id (String, Primary Key) - The unique ID of the Reel.
    
*   media\_type (String) - video, carousel, image.
    
*   compiled\_context (Text) - The complete markdown summary of the video.
    
*   vector\_embedding (Vector) - pgvector embedding of the context for semantic similarity searches.
    
*   processed\_at (Timestamp)
    

**Table: chat\_history**

*   message\_id (String, Primary Key)
    
*   ig\_user\_id (Foreign Key -> users)
    
*   ig\_post\_id (Foreign Key -> posts\_cache)
    
*   role (String) - user or assistant.
    
*   content (Text)
    
*   timestamp (Timestamp)
    

7\. Development & Deployment Pipeline
-------------------------------------

This shows how you intend to take the project from your local machine to production.

1.  **Local Dev:** Run FastAPI via Uvicorn. Spin up Redis and PostgreSQL using Docker Compose. Expose the port using Ngrok to receive Meta Webhooks.
    
2.  **Containerization:** Write a Dockerfile for the FastAPI app and a separate Dockerfile for the Celery worker.
    
3.  **Hosting Infrastructure:** Deploy to a cloud provider (e.g., Render, Railway, or AWS ECS) where the web server, worker, and Redis cluster can scale independently.
