# Building an AI Personal Assistant with Supabase: Persistent Memory & Autonomous Intelligence

Large Language Models excel at turning raw text into structured data, though they struggle to retrieve that structured data accurately across long sessions. In this post we'll use this strength and combine it with a Postgres database, along with a few other tools, to create a personalised Assistant with a long term memory.

At a high level, the system's flexibility is created by combining these core building blocks: An LLM owned database schema through an execute_sql tool, scheduled tasks for autonomy, web searches for real-time information, and MCP integrations for extended actions to may integrate with external tools.

See it at work in the video below.

<video className="rounded-sm m-0" autoPlay loop muted>
  <source
    src="https://xguihxuzqibwxjnimxev.supabase.co/storage/v1/object/public/videos/marketing/blog/natural-db/natural-db-demo-combined.mp4"
    type="video/mp4"
  />
</video>

## Core Pieces

### Scoped Database Control

The assistant uses a dedicated PostgreSQL schema called `memories` to store all of its structured data. To ensure security, the LLM operates under a specific role, `memories_role`, which is granted permissions only within this schema.

- **Scoped Schema**: The LLM can create tables, store data, and perform operations exclusively within the `memories` schema by calling an execute_sql tool
- **System Table Protection**: All other schemas, including `public`, are inaccessible to the LLM.

### Messages Context

Three complementary memory types maintain conversation continuity:

- **Message History (Short-term Memory)**: Maintains a chronological list of recent messages for immediate context
- **Semantic Memory (Vector Search using pgvector)**: Stores conversation embeddings using pgvector for fuzzy concept retrieval ("that productivity thing we talked about last month")
- **Structured Memory (SQL Data)**: Stores concrete facts in LLM-created tables for precise queries ("How much did I spend on coffee last quarter?")

### Scheduled Prompts

The system achieves autonomy through scheduled prompts which are powered by pg_cron through a dedicated tool. Scheduled prompts call the same natural-db endpoints and can therefore use all the same tools.

**Example**: "Every Sunday at 6 PM, analyze my portfolio performance and research market trends"

1.  **A cron job** executes a stored prompt at the scheduled time.
2.  **The secure tool** uses elevated permissions (`service_role`) to retrieve data from anywhere in your database, like current portfolio holdings.
3.  **Web search** is triggered to find relevant market news and competitor analysis.
4.  **Database storage** in the `memories` schema accumulates weekly performance data.
5.  **MCP integration** sends a personalized report via Zapier.
6.  **Memory building** enables future queries like "How has my portfolio performed compared to market trends?"

### Web Search

Real-time information gathering with intelligent storage:

```sql
-- Auto-generated from web search results
CREATE TABLE research_findings (
  topic TEXT,
  source_url TEXT,
  key_insights TEXT[],
  credibility_score INTEGER,
  search_date TIMESTAMPTZ DEFAULT NOW()
);
```

When you ask about current information, findings are structured and stored for future reference. Months later, "What were those Spanish apps I researched?" provides exact details with sources.

### Zapier MCP Integration

Through Zapier's MCP integration, your assistant can:

- Read/send emails (Gmail, Outlook)
- Manage calendar events
- Update spreadsheets
- Send notifications (Slack, Discord, SMS)
- Create tasks (Trello, Asana, Notion)
- Control smart home devices

### Input/Output Integration

The system uses Telegram as the default interface, implemented as an edge function with webhook support for real-time messaging. All input/output is processed through the `natural-db` function for consistent behavior.

### Self-Evolving System Prompt

The assistant maintains two behavioral layers:

- **Base Behavior**: Core functionality (database operations, scheduling, web search) remains consistent via a constant system prompt
- **Personalized Behavior**: Communication style and preferences that evolve based on user feedback which can be changed via a dedicated tool and stored in a public.system_prompts table

When you say "be more formal" or "address me by name," these preferences are stored with version history and persist across all conversations, creating a personalized experience.

## Use Cases

### Run Tracking

![Run tracking dashboard showing activity history and statistics](/images/blog/2025-06-10-natural-db/runs.png)

**Setup**: Track running activities and maintain consistent training schedule

1. **Database storage** creates `runs` table to store distance, duration, route, weather conditions, and personal notes for each run
2. **Daily reminders** via cron check last run date and send personalized Telegram notifications with previous run details to encourage consistency
3. **Run logging** allows users to record new runs through Telegram, automatically calculating pace and updating personal records
4. **Monthly summaries** via cron analyze running patterns, highlight achievements, and suggest training adjustments based on progress

### Personal Recipe & Meal Planning

**Setup**: "Track what I cook, suggest meals based on dietary preferences and ingredients I have"

1. **Database storage** creates `recipes`, `ingredients`, `meal_history`, and `meal_ratings` tables to track cooking experiences, dietary restrictions, ingredient preferences, and meal satisfaction
2. **Web search** finds new recipes based on available ingredients and dietary goals
3. **Database lookup** analyzes cooking patterns, favorite cuisines, nutritional balance, and meal ratings
4. **Cron trigger** runs weekly to suggest meal prep plans and grocery lists
5. **Telegram notifications** sends daily meal suggestions, cooking reminders, weekly grocery lists based on planned meals, and evening meal rating prompts
6. **Daily rating system** collects user feedback on each meal through Telegram, storing ratings and comments to improve future meal suggestions

### Company Feedback Analysis

**Setup**: "Every morning, pull the latest support tickets via MCP, analyze them, store the findings in a `feedback` table with tags, and give me a weekly summary."

1.  **Cron Trigger**: A `pg_cron` job runs every morning to initiate the ticket analysis workflow.
2.  **MCP Integration**: Connects to your support system (e.g., Zendesk, Intercom) to fetch new tickets.
3.  **AI Analysis & Storage**: The assistant processes each ticket, identifies key themes, sentiment, and product areas, then stores this structured data with tags in a `feedback` table.
4.  **Weekly Summary**: Every Friday, another cron job generates a summary of the week's feedback, highlighting top issues, sentiment trends, and feature requests.
5.  **Group Email**: The weekly summary is delivered as a concise report via Zapier's Gmail MCP, keeping your team informed of the customer's voice.

### Interest-Based Article Bookmarker

**Setup**: "Track articles about AI and climate change, remind me of important ones I haven't read"

1. **Database storage** creates `articles` table to store article metadata, user interests, read status, and relevance score
2. **Web search** daily finds new articles matching user interests
3. **Database lookup** analyzes reading patterns and article engagement
4. **Cron trigger** runs weekly to identify top unread articles by relevance
5. **Telegram notifications** sends personalized weekly digest with must-read articles based on interests

## Implementation Guide

### Prerequisites

- Supabase account (free tier sufficient)
- OpenAI API key
- Telegram bot token
- Zapier account (optional)

### Optional: Using the CLI

If you prefer the command line, you can use the Supabase CLI to set up your database and Edge Functions. This replaces **Step 1** and **Step 2**.

1.  **Clone the repository**.
    ```bash
    git clone https://github.com/supabase-community/natural-db.git
    cd natural-db
    ```
2.  **Log in to the Supabase CLI and link your project**.
    Create a new project on the [Supabase Dashboard](https://supabase.com/dashboard), then run:
    ```bash
    supabase login
    supabase link --project-ref <YOUR-PROJECT-ID>
    ```
3.  **Push the database schema**.
    ```bash
    supabase db push
    ```
4.  **Deploy Edge Functions**.
    ```bash
    supabase functions deploy --no-verify-jwt
    ```

After completing these steps, you can proceed to **Step 3: Telegram Bot**.

### Step 1: Database Setup

Run the migration SQL in your Supabase SQL editor: [migration.sql](https://github.com/supabase-community/natural-db/blob/main/supabase/migrations/001_create_initial_schema.sql)

- Sets up required extensions like `pgvector` and `pg_cron`.
- Creates the `memories` schema for the assistant's data.
- Creates the `memories_role` with scoped permissions to the `memories` schema.
- Configures cron job scheduling.

### Step 2: Edge Functions

Create three functions in Supabase dashboard:

**natural-db**: Main AI brain handling all processing, database operations, scheduling, and tool integration

- [natural-db/index.ts](https://github.com/supabase-community/natural-db/blob/main/supabase/functions/natural-db/index.ts)
- [natural-db/db-utils.ts](https://github.com/supabase-community/natural-db/blob/main/supabase/functions/natural-db/db-utils.ts)

**telegram-input**: Webhook handler for incoming messages with user validation and timezone management

- [telegram-input/index.ts](https://github.com/supabase-community/natural-db/blob/main/supabase/functions/telegram-input/index.ts)

**telegram-outgoing**: Response formatter and delivery handler with error management

- [telegram-outgoing/index.ts](https://github.com/supabase-community/natural-db/blob/main/supabase/functions/telegram-outgoing/index.ts)

### Step 3: Telegram Bot

1. Create bot via [@BotFather](https://t.me/botfather)
2. Set webhook: `https://api.telegram.org/bot[TOKEN]/setWebhook?url=https://[PROJECT].supabase.co/functions/v1/telegram-input`

### Step 4: Environment Variables

Set the following environment variables in your Supabase project settings (Project Settings → Edge Functions):

##### Required Variables:

- `OPENAI_API_KEY`: Your OpenAI API key
- `TELEGRAM_BOT_TOKEN`: Bot token from @BotFather

##### Optional Variables:

- `OPENAI_MODEL`: OpenAI model to use (defaults to "gpt-4.1-mini")
- `TELEGRAM_WEBHOOK_SECRET`: Secret token for webhook validation
- `TELEGRAM_ALLOWED_USERNAMES`: Comma-separated list of allowed Telegram usernames
- `ZAPIER_MCP_URL`: MCP server URL for Zapier integrations

### Step 5: Test Integration

Try these commands with your bot:

- "Store my grocery budget as $400 monthly"
- "What's the weather today?" (web search)
- "Remind me to exercise every Monday at 7 AM"
- "Be more enthusiastic when I discuss hobbies" (personality)

## Input and Output Methods

The natural-db edge function is decoupled from how messages are received or sent out. This means you can interact with your AI companion through any channel you prefer:

- WhatsApp messages
- Email
- Slack
- Web interface
- Any other messaging platform

## Cost Considerations

Based on 10 messages per day (300 messages/month):

- **Supabase**: Free tier (500MB database, 2GB bandwidth) - $0/month
- **OpenAI GPT-4.1-mini**: $0.40 per 1M input tokens, $1.60 per 1M output tokens
  - Average 1200 input + 800 output tokens per message
  - Input: 300 messages × 1200 tokens × $0.40/1M = $0.144/month
  - Output: 300 messages × 800 tokens × $1.60/1M = $0.384/month
  - Total OpenAI: $0.53/month
- **Telegram**: Free API usage
- **Zapier**: Free tier (300 tasks/month) - $0/month
- **Vector Embeddings**: $0.02 per 1M tokens (text-embedding-3-small)
  - 300 messages × 1200 tokens × $0.02/1M = $0.0072/month

**Total monthly cost: ~$0.54**

## Make it your own

This experiment demonstrates the power of combining fundamental building blocks, with an LLM being one, to create something greater than the sum of its parts. My hope is this also inspires you to build and deploy your own personalized assistant in a way that still gives you control over the code and the data.

If you're interested in helping to evolve this as a starting template, please feel free to contribute via issues, discussions or pull requests.
