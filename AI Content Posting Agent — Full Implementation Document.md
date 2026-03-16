# AI Content Posting Agent — Full Implementation Document

## 1. Project Goal

Build a business AI agent that can:

* learn a business profile and brand voice
* store long-term business memory
* generate content for multiple channels
* decide what to post and when to post
* support human approval
* publish to multiple platforms
* recover safely from crashes using WAL
* avoid context-window problems using structured memory + retrieval

Primary target platforms:

* Blog / CMS
* Facebook
* LinkedIn
* News site / portal
* optional later: X, Instagram, Telegram, email newsletter

---

## 2. Product Vision

This is not just a scheduler.

It is an **AI Content Distribution Agent** that combines:

* business knowledge memory
* content planning
* post generation
* schedule optimization
* approval workflow
* multi-platform publishing
* analytics feedback loop

Core business value:

* reduce manual marketing work
* improve posting consistency
* reuse one content idea across many platforms
* keep brand voice stable
* improve engagement over time

---

## 3. MVP Scope

Build the first version with these features only:

### Must-have

1. business profile setup
2. brand voice and posting rules
3. content topic creation
4. AI post generation
5. manual approval before publish
6. scheduling
7. multi-platform publish jobs
8. write-ahead logging for reliable execution
9. session summary memory
10. simple analytics capture

### Platforms for MVP

* Blog / internal CMS
* LinkedIn
* Facebook

### Do not build in MVP

* full autonomous reply to comments
* competitor intelligence engine
* advanced A/B testing
* image generation pipeline
* complex campaign automation
* all platforms at once

---

## 4. Recommended Stack

### Main application

* Laravel 12+
* PHP 8.3+
* PostgreSQL
* Redis
* Laravel Queues / Horizon

### AI and retrieval

* OpenAI model API for generation/classification
* pgvector in PostgreSQL for semantic search
* optional Python microservice later for embedding-heavy workloads

### Infra

* Nginx
* Supervisor for queue workers
* S3-compatible object storage for media/assets

### Frontend/Admin

* Blade or Inertia/Vue
* Admin dashboard for approval, schedules, logs, analytics

---

## 5. High-Level Architecture

## Main modules

1. **Business Profile Module**

   * company information
   * tone
   * audience
   * business rules

2. **Memory Module**

   * structured facts
   * summaries
   * vector retrieval

3. **Prompt Builder Module**

   * prepares clean, task-specific prompts

4. **AI Content Engine**

   * generates posts, summaries, titles, variations

5. **Planner Module**

   * decides what content to create and when to publish

6. **Approval Module**

   * human review before publish

7. **Publishing Module**

   * dispatches platform jobs

8. **WAL + Recovery Module**

   * durable logging of all critical actions

9. **Analytics Module**

   * engagement data
   * best-time recommendations

---

## 6. Core Workflow

### Main flow

1. business sets profile
2. business connects platforms
3. user creates campaign or weekly goal
4. planner creates content plan
5. AI generates drafts
6. drafts go to approval queue
7. approved drafts are scheduled
8. publish jobs execute per platform
9. results are logged in WAL
10. analytics are collected
11. memory summaries and insights are updated

---

## 7. Database Design

## 7.1 Businesses

### `businesses`

* id
* name
* slug
* timezone
* website_url
* created_at
* updated_at

### `business_profiles`

* id
* business_id
* description
* industry
* target_audience_json
* goals_json
* created_at
* updated_at

### `brand_voice_profiles`

* id
* business_id
* tone
* style_rules_json
* banned_terms_json
* approved_cta_patterns_json
* created_at
* updated_at

### `platform_accounts`

* id
* business_id
* platform
* account_name
* external_account_id
* auth_status
* access_token_encrypted
* refresh_token_encrypted
* token_expires_at
* metadata_json
* created_at
* updated_at

---

## 7.2 Content

### `content_topics`

* id
* business_id
* title
* description
* source_type
* source_ref
* priority
* status
* created_at
* updated_at

### `content_items`

* id
* business_id
* topic_id
* content_type
* title
* body
* summary
* status
* ai_model
* ai_prompt_version
* source_memory_snapshot_json
* created_at
* updated_at

### `content_variants`

* id
* content_item_id
* platform
* title
* body
* status
* created_at
* updated_at

### `content_approvals`

* id
* content_variant_id
* reviewer_id
* status
* notes
* reviewed_at
* created_at
* updated_at

---

## 7.3 Scheduling and Publishing

### `posting_schedules`

* id
* business_id
* content_variant_id
* platform_account_id
* scheduled_for
* timezone
* status
* publish_attempts
* last_attempt_at
* created_at
* updated_at

### `published_posts`

* id
* posting_schedule_id
* platform
* external_post_id
* published_at
* external_url
* response_json
* created_at
* updated_at

---

## 7.4 WAL and Agent Runs

### `agent_runs`

* id
* business_id
* workflow_name
* status
* input_json
* output_json
* started_at
* completed_at
* created_at
* updated_at

### `agent_wal`

* id
* agent_run_id
* step_no
* event_type
* status
* idempotency_key
* payload_json
* created_at
* updated_at

### `outbox_jobs`

* id
* business_id
* agent_run_id
* job_type
* payload_json
* status
* available_at
* processed_at
* retry_count
* last_error
* created_at
* updated_at

---

## 7.5 Memory

### `memory_items`

* id
* business_id
* memory_type
* scope
* key
* value_json
* importance_score
* last_used_at
* expires_at
* created_at
* updated_at

### `session_summaries`

* id
* agent_run_id
* summary_text
* created_at
* updated_at

### `document_chunks`

* id
* business_id
* source_type
* source_id
* chunk_text
* embedding
* metadata_json
* created_at
* updated_at

---

## 7.6 Analytics

### `post_metrics`

* id
* published_post_id
* platform
* impressions
* clicks
* reactions
* comments
* shares
* ctr
* engagement_rate
* fetched_at
* created_at
* updated_at

### `analytics_insights`

* id
* business_id
* platform
* insight_type
* insight_value_json
* confidence_score
* created_at
* updated_at

---

## 8. Memory Architecture

Use 4 memory layers.

## 8.1 Working memory

For current task only.

Examples:

* requested platform
* selected topic
* last tool result
* current draft id

Do not persist all of this forever.

## 8.2 Session memory

Summarized memory for one run.

Example summary:

* 5 posts generated
* 3 approved
* LinkedIn best window selected for Tuesday morning

## 8.3 Long-term structured memory

Store business facts as rows, not chat text.

Examples:

* tone = professional
* avoid exaggerated claims
* preferred CTA = question-based
* audience = founders and SMEs

## 8.4 Retrieval memory

Use pgvector for:

* past posts
* old blog articles
* campaign summaries
* analytics notes

Before each model call, retrieve only the most relevant chunks.

---

## 9. Prompt Building Strategy

Never send full history.

Each prompt should include only:

1. system rules
2. current task
3. relevant business facts
4. top retrieved memory chunks
5. session summary
6. output format instructions

### Example prompt structure

#### System

You are a content generation agent. Follow brand voice strictly. Never invent business facts.

#### Current task

Generate 3 LinkedIn post drafts for next week about AI workflow automation.

#### Brand facts

* tone: professional and practical
* audience: SME owners and startup founders
* avoid: political claims, fake promises

#### Retrieved context

* top post last month used short hook + 3 bullets + CTA question
* best-performing topic: reducing manual work

#### Session summary

Campaign goal is lead generation for the blog and LinkedIn channels.

#### Output format

Return JSON array with title, body, CTA, hashtags.

---

## 10. WAL Strategy

## Rule

No external side effect without durable log first.

### WAL event types for MVP

* TASK_RECEIVED
* MODEL_RESPONSE_ACCEPTED
* POST_SCHEDULED
* TOOL_CALL_PREPARED
* TOOL_CALL_EXECUTED
* TOOL_CALL_FAILED
* RUN_COMPLETED

### Why this matters

If the system crashes after posting to LinkedIn but before saving state, WAL helps recovery.

### Required fields

* agent_run_id
* step_no
* event_type
* status
* idempotency_key
* payload_json

---

## 11. Idempotency Strategy

Each external action must have a unique idempotency key.

### Example

`run_25_step_7_linkedin_publish`

Use this key:

* in WAL row
* in publish job payload
* in external request metadata if platform supports it

If retry happens, the system can detect duplicate execution.

---

## 12. Laravel Implementation Steps

## Step 1 — Create core migrations

Build migrations for:

* businesses
* business_profiles
* brand_voice_profiles
* platform_accounts
* content_topics
* content_items
* content_variants
* content_approvals
* posting_schedules
* published_posts
* agent_runs
* agent_wal
* outbox_jobs
* memory_items
* session_summaries
* document_chunks
* post_metrics
* analytics_insights

---

## Step 2 — Build Eloquent models and relationships

Example main relationships:

* Business hasOne BusinessProfile
* Business hasOne BrandVoiceProfile
* Business hasMany PlatformAccounts
* Business hasMany ContentTopics
* ContentTopic hasMany ContentItems
* ContentItem hasMany ContentVariants
* ContentVariant hasOne ContentApproval
* ContentVariant hasMany PostingSchedules
* PostingSchedule hasOne PublishedPost
* AgentRun hasMany AgentWal

---

## Step 3 — Create service classes

Recommended services:

* `BusinessProfileService`
* `MemoryService`
* `PromptBuilderService`
* `AiContentService`
* `SchedulingService`
* `ApprovalService`
* `PublishingService`
* `AgentWalService`
* `RecoveryService`
* `AnalyticsService`

Keep controllers thin.

---

## Step 4 — Business setup flow

Build admin pages for:

* business profile
* tone settings
* banned terms
* platform account connection
* posting preferences

When saved, also create/update memory items.

Examples of memory rows:

* memory_type = `fact`, key = `tone`
* memory_type = `rule`, key = `avoid_terms`
* memory_type = `preference`, key = `preferred_platforms`

---

## Step 5 — Topic intake

Allow topic creation from:

* manual entry
* imported blog article
* imported news link
* uploaded note/document

Each topic becomes a possible content source.

Also chunk long source content into `document_chunks` and create embeddings.

---

## Step 6 — Build embedding and chunking pipeline

For source content:

1. clean text
2. split into chunks
3. generate embeddings
4. save to `document_chunks`

### Chunking guidelines

* 400 to 800 tokens per chunk
* keep section boundaries where possible
* store metadata like source title, source url, topic id

---

## Step 7 — Build prompt builder

`PromptBuilderService` should:

1. receive task type
2. fetch structured facts
3. retrieve top memory chunks
4. include session summary
5. render final prompt

### Task-specific prompt builders

* `buildForLinkedInPost()`
* `buildForFacebookPost()`
* `buildForBlogSummary()`
* `buildForScheduleRecommendation()`
* `buildForCampaignPlan()`

---

## Step 8 — AI content generation

### Generate content item

Flow:

1. create `agent_runs` row
2. WAL: TASK_RECEIVED
3. build prompt
4. call model
5. validate output schema
6. save draft to `content_items` / `content_variants`
7. WAL: MODEL_RESPONSE_ACCEPTED
8. create session summary

Use JSON schema validation for model outputs whenever possible.

---

## Step 9 — Approval workflow

### Approval states

* draft
* pending_review
* approved
* rejected
* revision_requested

When AI generates content, create variants with `pending_review`.

Admin can:

* approve
* edit and approve
* reject
* request new variation

When approved, create `posting_schedules` record.

---

## Step 10 — Scheduling engine

Start with a simple rule-based engine.

### Inputs

* platform
* business timezone
* posting frequency rules
* historical best times
* business preference

### MVP logic example

* LinkedIn: Tue–Thu, 9 AM–11 AM
* Facebook: Wed–Fri, 6 PM–8 PM
* Blog: Mon–Thu, 8 AM–10 AM

Later, replace this with analytics-driven scoring.

Store selected publish time in `posting_schedules`.

WAL: POST_SCHEDULED

---

## Step 11 — Outbox job creation

When a schedule is approved and ready:

1. write WAL entry
2. create `outbox_jobs` row
3. commit transaction

Do not publish directly inside controller.

### Job types

* `publish_blog_post`
* `publish_linkedin_post`
* `publish_facebook_post`
* `sync_post_metrics`

---

## Step 12 — Queue workers

Create Laravel jobs:

* `PublishBlogPostJob`
* `PublishLinkedInPostJob`
* `PublishFacebookPostJob`
* `SyncPostMetricsJob`
* `RecoverIncompleteRunsJob`

Use Horizon or Supervisor-managed workers.

---

## Step 13 — Publishing flow with WAL

### Example publish flow

1. worker loads outbox job
2. WAL: TOOL_CALL_PREPARED
3. call platform API
4. on success save `published_posts`
5. WAL: TOOL_CALL_EXECUTED
6. mark schedule status = published
7. mark outbox job processed

### On failure

1. WAL: TOOL_CALL_FAILED
2. store error in `outbox_jobs.last_error`
3. increment retry_count
4. reschedule if retry allowed

---

## Step 14 — Recovery service

`RecoveryService` should periodically:

1. find incomplete agent runs
2. inspect latest WAL events
3. find prepared-but-unresolved tool actions
4. check whether external post exists using idempotency key or external references
5. mark success or requeue safely

This makes the system reliable after crash or worker failure.

---

## Step 15 — Platform adapters

Create one adapter per platform.

### Interface example

* `publish(array $payload): array`
* `fetchMetrics(string $externalPostId): array`
* `validateConnection(): bool`

### Adapters

* `BlogPublisherAdapter`
* `LinkedInPublisherAdapter`
* `FacebookPublisherAdapter`

Keep API-specific logic out of jobs and controllers.

---

## Step 16 — Analytics sync

After publish, schedule metrics sync jobs.

Capture:

* impressions
* clicks
* reactions
* comments
* shares
* engagement rate

Store into `post_metrics`.

Then generate higher-level insights into `analytics_insights`.

Example insight:

* LinkedIn educational posts perform best on Tuesday morning

---

## Step 17 — Update memory from analytics

When stable patterns appear, write them into memory.

Examples:

* best_time_linkedin = Tuesday 10:00 AM
* best_post_style = short hook + bullets + CTA question
* low-performing_topic = generic motivational posts

These memory items are later used by the planner and prompt builder.

---

## Step 18 — Campaign planner

Build a planner that converts goal into tasks.

### Input

* business goal
* target platform(s)
* date range
* target topic(s)

### Output

* list of topics
* recommended formats
* schedule suggestions

Start with rule-based planning + model assistance.

Do not let the model directly control publishing in MVP.

---

## Step 19 — Session summarization

At the end of each run, create a compact summary.

Example:

* generated 4 drafts
* approved 2 LinkedIn posts
* scheduled one blog post for Monday 9:00 AM
* Facebook drafts need revision

Store in `session_summaries`.

This becomes short-term reusable memory without keeping all raw context.

---

## Step 20 — Admin dashboard

Build pages for:

* businesses
* profile and voice
* platform connections
* content topics
* drafts and approvals
* schedules calendar
* published posts
* metrics dashboard
* WAL / run logs
* failed jobs / recovery

For business trust, WAL visibility is very important.

---

## 13. Example Laravel Service Design

## AgentWalService

Responsibilities:

* append WAL entries
* fetch unresolved tool steps
* find run timeline

## MemoryService

Responsibilities:

* upsert facts
* save summaries
* rank memory relevance
* retrieve top chunks

## PromptBuilderService

Responsibilities:

* compile task-specific prompt input
* inject structured facts
* inject summary
* inject retrieved evidence

## AiContentService

Responsibilities:

* send request to model
* validate response structure
* normalize output
* persist drafts

## PublishingService

Responsibilities:

* dispatch correct platform adapter
* attach idempotency key
* return normalized publish response

## RecoveryService

Responsibilities:

* inspect incomplete runs
* resolve uncertain publish state
* requeue safe operations

---

## 14. Suggested Folder Structure

```text
app/
  Actions/
  DTOs/
  Enums/
  Jobs/
  Models/
  Services/
    AI/
    Memory/
    Publishing/
    Recovery/
  Support/
    PlatformAdapters/
      Contracts/
      Blog/
      Facebook/
      LinkedIn/
```

Use DTOs for clean internal contracts.

---

## 15. Important Enums

### Content status

* draft
* pending_review
* approved
* rejected
* scheduled
* published
* failed

### Agent run status

* pending
* running
* completed
* failed
* recovery_pending

### WAL status

* pending
* in_progress
* success
* failed
* compensated

### Platform

* blog
* facebook
* linkedin
* news_site

---

## 16. Security and Stability Rules

1. encrypt tokens at rest
2. never store raw secrets in logs
3. redact sensitive payload values in WAL if needed
4. require approval before publish in MVP
5. validate all model output before using it
6. use idempotency keys for external actions
7. put rate-limit handling in platform adapters
8. isolate publishing jobs from UI request lifecycle

---

## 17. Cost-Control Strategy

AI cost can grow fast.

Use these controls:

* summarize session history
* retrieve top 3–5 relevant chunks only
* store reusable generated drafts
* use small/cheap model for classification and extraction
* use stronger model only for high-value generation
* cache embedding results for unchanged content

---

## 18. Testing Strategy

## Unit tests

* prompt builder outputs expected prompt sections
* memory ranking logic works
* scheduling rules select valid time windows
* WAL service appends correct events
* adapter payload transformers work

## Integration tests

* content generation flow creates drafts and WAL rows
* approval creates schedule
* outbox job publishes and stores result
* recovery resolves incomplete publish action

## End-to-end tests

* business setup → topic → draft → approval → publish → metrics sync

---

## 19. Rollout Plan

## Phase 1 — Foundation

* core schema
* business setup
* WAL
* memory base layer
* draft generation

## Phase 2 — Scheduling + Approval

* approval queue
* schedule calendar
* outbox publishing jobs

## Phase 3 — Analytics

* metrics sync
* insight generation
* best-time learning

## Phase 4 — Smarter Planning

* campaign planner
* topic suggestion
* automated repurposing

## Phase 5 — Advanced AI Agent

* limited autonomous planning
* optional auto-post for trusted accounts
* A/B testing suggestions
* human-in-the-loop exceptions

---

## 20. Future Features

After the MVP is stable, add:

1. competitor topic analysis
2. automatic post repurposing from blog to social variants
3. campaign goals and content calendars
4. AI-assisted image caption generation
5. multi-language generation
6. approval by role/team
7. recommendation scoring for best content type
8. trend ingestion from selected sources
9. advanced analytics dashboard
10. safe auto-post mode for high-confidence content

---

## 21. Stability Assessment

### Is this product stable for the future?

Yes, if built with these rules:

* model is not the only memory
* state is stored in DB
* side effects are protected by WAL
* external actions use idempotency
* prompts are task-specific
* publishing stays behind approval until trust is built

### Main long-term risks

* social platform API changes
* AI content quality inconsistency
* token cost growth
* duplicated posts if idempotency is ignored
* noisy prompts if memory retrieval is weak

### Main long-term protection

* adapter architecture
* WAL + recovery
* structured memory + RAG
* approval workflow
* analytics feedback loop

---

## 22. Recommended Build Order for You

1. core DB schema
2. business profile + voice setup
3. memory items + session summaries
4. content topic intake
5. prompt builder
6. AI draft generation
7. approval UI
8. schedule creation
9. WAL + outbox
10. LinkedIn publish adapter
11. Facebook publish adapter
12. blog publish adapter
13. metrics sync
14. analytics insights
15. recovery worker

This order gives fastest business value with lower technical risk.

---

## 23. Final Recommendation

For your first production version, build this exact product shape:

**AI-assisted multi-platform content agent with human approval**

That is much safer than a fully autonomous AI poster.

### MVP promise

* brand-aware drafts
* platform-specific formatting
* scheduling help
* reliable publish flow
* analytics learning

### Technical promise

* WAL for reliability
* structured memory for context control
* retrieval for large knowledge
* adapters for multi-platform support

This is a strong business product direction and a stable technical foundation.
