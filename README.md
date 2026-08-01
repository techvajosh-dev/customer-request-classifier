Customer Request Classifier

An n8n workflow that reads incoming support email, classifies it with an AI agent, logs it to Airtable, assigns it to the right person, and either auto-replies or escalates to Slack. Ships with a token-gated dashboard backend for reporting.

Show Image

The problem

Support inboxes arrive as an undifferentiated pile. Someone has to open each message, work out whether it's billing, a bug, or a sales question, decide how urgent it is, find the right owner, and log it somewhere trackable. It's an hour a day of work that follows fixed rules — exactly the kind of thing that should be automated, but only if the automation is careful enough to be trusted with real customer mail.

How it works

Intake. A Gmail trigger polls every minute. An IF node filters out auto-replies, bounces, and no-reply senders before anything else runs, so machine noise never reaches the AI step.

Deduplication. Every message is checked against Airtable by message ID before processing. Without this, a re-poll after a failed run creates duplicate records and, worse, duplicate replies to a customer. Duplicates terminate at a NoOp.

Classification. An AI agent categorises the request, scores urgency and confidence, writes a short reasoning note, drafts a response, and flags whether a human should look at it. A structured output parser enforces the response schema so downstream nodes get predictable JSON instead of prose.

Validation. A code node validates and enriches the agent output before anything is written. Category values are checked against the allowed set and confidence is bounds-checked — an LLM returning a plausible-looking but invalid category shouldn't be able to corrupt the Airtable records.

Assignment. The workflow looks up the Agent Directory table and resolves the owner for that category, including their Slack handle for mentions.

Routing. High-confidence, low-risk requests get an automatic Gmail reply and are marked auto-handled. Anything flagged for human review, or below the confidence threshold, posts to Slack with the full context and draft response so a person can review and send.

Error handling. A separate error trigger catches failures anywhere in the workflow, writes the failed node, message, timestamp, and input payload to an errors table, and posts to Slack. Silent failures are the main risk in an automation that touches customer mail — this makes them loud.

Dashboard backend

A second workflow (dashboard-backend.json) exposes the Airtable data as a JSON API for a reporting front end. A webhook receives the request, checks a token in the query string, pulls from the requests, agents, and errors tables, and computes metrics — automation rate, category breakdown, per-agent workload, recent activity. Invalid tokens get a 401 from a separate response node.

Stack

n8n · Airtable · Gmail · Slack · OpenAI (via LangChain agent node)

Setup
Import both JSON files into your n8n instance
Create credentials for Gmail, Airtable, Slack, and your LLM provider, then reconnect each node — credential IDs are stripped from these exports
Create three Airtable tables: Customer Requests, Agent Directory, Workflow Errors
Replace the placeholders below with your own values
Placeholder	Where
YOUR_AIRTABLE_BASE_ID	all Airtable nodes
YOUR_CUSTOMER_REQUESTS_TABLE_ID	intake, dedup, status nodes
YOUR_AGENT_DIRECTORY_TABLE_ID	Lookup Agent
YOUR_WORKFLOW_ERRORS_TABLE_ID	Log Error to Airtable
YOUR_SLACK_CHANNEL_ID	Slack Alert, Error Slack Alert
YOUR_DASHBOARD_TOKEN	Valid Token? node in the dashboard backend
Adjust the categories in the agent's system prompt to match your own taxonomy
Set the workflow's error workflow to point at the error trigger
Notes

The dedup check matters more than it looks. It's the difference between a failed run being a non-event and a customer getting the same reply twice.

Validating agent output before writing is the other load-bearing piece. Structured output parsing gets you well-formed JSON, not correct JSON — the schema can be satisfied by a category that doesn't exist in your Airtable.

The webhook currently allows all origins. Restrict this to your dashboard's domain before running it anywhere real.

Built by Joshua Clyde Manuel — AI automation and workflow engineering.
