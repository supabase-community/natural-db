# Building an AI Personal Assistant with Supabase: Persistent Memory & Autonomous Intelligence

Large Language Models excel at natural language understanding but struggle with maintaining structured data across conversations. This system enhances LLMs by combining their natural language processing with PostgreSQL storage, creating an AI that maintains precise, queryable records over time. The assistant converts conversations into structured database entries, enabling reliable data retrieval and analysis through database operations, scheduled prompts, web searches, and MCP integrations.

The system's power comes from how its core components work together: database operations store structured data, scheduled tasks create autonomy, web searches gather real-time information, and MCP integrations enable real-world actions. This creates a flexible foundation that starts simple (like tracking expenses) and can evolve into complex workflows (like automated investment monitoring). For example, a scheduled task might analyze your portfolio, trigger web searches for market trends, update your database with new insights, and send personalized reports through Zapier—all while maintaining organized, queryable data that improves with each cycle.

<video className="rounded-sm m-0" autoPlay loop muted>
  <source
    src="https://xguihxuzqibwxjnimxev.supabase.co/storage/v1/object/public/videos/marketing/blog/natural-db/natural-db-demo-combined.mp4"
    type="video/mp4"
  />
</video>

## Core Pieces

### Scoped Database Control

Each chat operates in a completely isolated PostgreSQL schema (`chat_{chat_id}`), providing bulletproof security:

- **Private Schemas**: LLM can create tables, store data, and perform operations without accessing other users' information
- **System Table Protection**: Critical system tables remain in the `public` schema, completely inaccessible to the LLM
- **Automatic Schema Creation**: New chats get properly configured private schemas with restricted permissions
- **Complete Data Separation**: Chat A's tables are invisible to Chat B, preventing any cross-contamination

```sql
-- Auto-generated from natural language in your private schema
create table expenses (
  id uuid primary key,
  amount decimal,
  category text,
  date timestamptz default NOW()
);

insert into expenses (amount, category, store, date, note)
values (47.00, 'groceries', 'Whole Foods', '2024-01-15', 'Monthly budget target: $400');
```

### Messages Context

Two complementary memory types maintain conversation continuity:

- **Semantic Memory (Vector Search)**: Stores conversation embeddings using pgvector for fuzzy concept retrieval ("that productivity thing we talked about last month")
- **Structured Memory (SQL Data)**: Stores concrete facts in LLM-created tables for precise queries ("How much did I spend on coffee last quarter?")

### Scheduled Prompts

The system's autonomy emerges through scheduled prompts via pg_cron that can make use of all other pieces:

**Example**: "Every Sunday at 6 PM, analyze my portfolio performance and research market trends"

1. **Cron trigger** executes with stored prompt
2. **Database lookup** retrieves your current portfolio holdings and historical performance
3. **Web search** finds relevant market news and competitor analysis
4. **Database storage** accumulates weekly performance data and market insights
5. **MCP integration** sends personalized report via Zapier with portfolio highlights and recommendations
6. **Memory building** enables future queries like "How has my portfolio performed compared to market trends?"

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

These aren't one-off actions—they're integrated into the data collection and analysis workflow.

### Input/Output Integration

The system uses Telegram as the default interface, implemented as an edge function with webhook support for real-time messaging. All input/output is processed through the `natural-db` function for consistent behavior.

### Self-Evolving System Prompt

The assistant maintains two behavioral layers:

- **Base Behavior**: Core functionality (database operations, scheduling, web search) remains constant
- **Personalized Behavior**: Communication style and preferences that evolve based on user feedback

When you say "be more formal" or "address me by name," these preferences are stored with version history and persist across all conversations, creating a truly personalized AI companion.

## Code Ownership & Extensibility

As the codebase owner, you have complete control over your assistant's capabilities, allowing you to modify base behavior by customizing system prompts to adjust personality and expertise or create custom edge functions for specific needs.

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

### Travel Planning & Experience Tracking

**Setup**: "Help me research destinations and track my travel experiences"

1. **Database storage** creates `destinations`, `trip_plans`, and `travel_experiences` tables to store research findings, itineraries, and post-trip reflections
2. **Web search** finds destination information, flight deals, and local recommendations based on travel preferences
3. **Database lookup** analyzes past trip satisfaction, budget patterns, and preferred activities
4. **Cron trigger** runs monthly to suggest new destinations and seasonal travel deals
5. **MCP integration** automatically adds trip dates to calendar and sends pre-trip reminders with personalized packing lists and local tips

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

### Step 1: Database Setup

Run the migration SQL in your Supabase SQL editor: [migration.sql](https://github.com/supabase-community/natural-db/blob/main/supabase/migrations/001_create_initial_schema.sql)

- Sets up extensions (pgvector, pg_cron)
- Creates system tables with proper permissions
- Configures cron job scheduling

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

Note: Costs scale linearly with usage. At 100 messages/day (~3K messages/month), expect ~$5.40/month.

## Summary

This system leverages what LLMs do best—understanding natural language and transforming it into structured data—while solving their biggest weakness: memory persistence. By combining this with PostgreSQL's querying power, cron scheduling, and external tool integration, you get an AI that genuinely gets smarter over time.

**Key advantages**:

- **Persistent Memory**: Each interaction builds structured, queryable knowledge
- **Complete Privacy**: Isolated database schemas ensure bulletproof data security
- **Autonomous Intelligence**: Scheduled operations create self-improving analysis loops
- **Real-world Integration**: Actions across email, calendar, and hundreds of tools
- **Adaptive Personality**: Evolving communication style based on user preferences

This isn't just another chatbot—it's a persistent AI companion that accumulates knowledge and takes autonomous action, becoming more valuable the longer you use it.
