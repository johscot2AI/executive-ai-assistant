# Executive AI Assistant

Executive AI Assistant (EAIA) is an AI agent that attempts to do the job of an Executive Assistant (EA).

For a hosted version of EAIA, see documentation [here](https://mirror-feeling-d80.notion.site/How-to-hire-and-communicate-with-an-AI-Email-Assistant-177808527b17803289cad9e323d0be89?pvs=4).

Table of contents

- [General Setup](#general-setup)
  - [Env](#env)
  - [Credentials](#env)
  - [Configuration](#configuration)
- [Run locally](#run-locally)
  - [Setup EAIA](#set-up-eaia-locally)
  - [Ingest emails](#ingest-emails-locally)
  - [Connect to Agent Inbox](#set-up-agent-inbox-with-local-eaia)
  - [Use Agent Inbox](#use-agent-inbox)
- [Run in production (LangGraph Cloud)](#run-in-production--langgraph-cloud-)
  - [Setup EAIA on LangGraph Cloud](#set-up-eaia-on-langgraph-cloud)
  - [Ingest manually](#ingest-manually)
  - [Set up cron job](#set-up-cron-job)

## General Setup

### Env

1. Fork and then clone this repo. Note: make sure to fork it first, as in order to deploy this you will need your own repo.
2. **Python Version**: Use Python 3.11.x (required due to PyO3/tiktoken compatibility issues with Python 3.13+)
3. Install Poetry if not already installed: `pip install poetry`
4. Set up the environment:
   ```bash
   poetry install
   ```
   
   **Note**: If you encounter build errors with tiktoken on Python 3.13+, run:
   ```bash
   export PYO3_USE_ABI3_FORWARD_COMPATIBILITY=1
   poetry install
   ```

### Set up credentials

1. Export OpenAI API key (`export OPENAI_API_KEY=...`)
2. Export Anthropic API key (`export ANTHROPIC_API_KEY=...`)
3. Enable Google
   1. [Enable the API](https://developers.google.com/gmail/api/quickstart/python#enable_the_api)
      - Enable Gmail API if not already by clicking the blue button `Enable the API`
   2. [Authorize credentials for a desktop application](https://developers.google.com/gmail/api/quickstart/python#authorize_credentials_for_a_desktop_application)
  
> Note: If you're using a personal email (non-Google Workspace), select "External" as the User Type in the OAuth consent screen. With "External" selected, you must add your email as a test user in the Google Cloud Console under "OAuth consent screen" > "Test users" to avoid the "App has not completed verification" error. The "Internal" option only works for Google Workspace accounts.

4. Download the client secret. After that, run these commands:
5. `mkdir -p eaia/.secrets` - This will create a folder for secrets
6. `mv ${PATH-TO-CLIENT-SECRET.JSON} eaia/.secrets/secrets.json` - This will move the client secret you just created to that secrets folder
7. Set up Gmail OAuth token:
   ```bash
   poetry run python scripts/setup_gmail.py
   ```
   This will:
   - Open a browser window for Gmail authentication
   - Prompt you to sign in with your Gmail account
   - Request permissions for Gmail and Calendar access
   - Generate `eaia/.secrets/token.json` automatically
   
   **Complete the OAuth flow** by:
   - Clicking the authorization URL that appears in the terminal
   - Signing in with your Gmail account
   - Granting the requested permissions
   - The script will automatically capture the token when you're redirected

8. Set up environment variables:
   ```bash
   export OPENAI_API_KEY="your_openai_key"
   export ANTHROPIC_API_KEY="your_anthropic_key"
   export LANGSMITH_API_KEY="your_langsmith_key"
   ```
   
   **Optional**: Create a `.env` file in the project root for persistent environment variables:
   ```bash
   echo "OPENAI_API_KEY=your_openai_key" >> .env
   echo "ANTHROPIC_API_KEY=your_anthropic_key" >> .env
   echo "LANGSMITH_API_KEY=your_langsmith_key" >> .env
   ```

### Configuration

The configuration for EAIA can be found in `eaia/main/config.yaml`. Every key in there is required. 

**Important**: Update the configuration with your personal details:
- Replace all email addresses with your Gmail account
- Update name, background, timezone, and preferences
- Customize triage rules for your specific needs (work vs personal emails)
- Update any company-specific information (e.g., replace LangChain references with your organization)

These are the configuration options:

- `email`: Email to monitor and send emails as. This should match the credentials you loaded above.
- `full_name`: Full name of user
- `name`: First name of user
- `background`: Basic info on who the user is
- `timezone`: Default timezone the user is in
- `schedule_preferences`: Any preferences for how calendar meetings are scheduled. E.g. length, name of meetings, etc
- `background_preferences`: Any background information that may be needed when responding to emails. E.g. coworkers to loop in, etc.
- `response_preferences`: Any preferences for what information to include in emails. E.g. whether to send calendly links, etc.
- `rewrite_preferences`: Any preferences for the tone of your emails
- `triage_no`: Guidelines for when emails should be ignored
- `triage_notify`: Guidelines for when user should be notified of emails (but EAIA should not attempt to draft a response)
- `triage_email`: Guidelines for when EAIA should try to draft a response to an email

## Run locally

You can run EAIA locally.
This is useful for testing it out, but when wanting to use it for real you will need to have it always running (to run the cron job to check for emails).
See [this section](#run-in-production--langgraph-cloud-) for instructions on how to run in production (on LangGraph Cloud)

### Set up EAIA locally

1. Install development server:
   ```bash
   poetry add --group dev "langgraph-cli[inmem]"
   ```
   Or if you prefer to install globally:
   ```bash
   pip install -U "langgraph-cli[inmem]"
   ```

2. Run development server:
   ```bash
   langgraph dev
   ```

### Ingest Emails Locally

Let's now kick off an ingest job to ingest some emails and run them through our local EAIA.

Leave the `langgraph dev` command running, and open a new terminal. From there, get back into this directory and virtual environment. To kick off an ingest job, run:

```shell
poetry run python scripts/run_ingest.py --minutes-since 120 --rerun 1 --early 0
```

This will ingest all emails in the last 120 minutes (`--minutes-since`). It will NOT break early if it sees an email it already saw (`--early 0`) and it will
rerun ones it has seen before (`--rerun 1`). It will run against the local instance we have running.

### Set up Agent Inbox with Local EAIA

After we have [run it locally](#run-locally), we can interract with any results.

1. Go to [Agent Inbox](https://dev.agentinbox.ai/)
2. Connect this to your locally running EAIA agent:
   1. Click into `Settings`
   2. Input your LangSmith API key.
   3. Click `Add Inbox`
      1. Set `Assistant/Graph ID` to `main`
      2. Set `Deployment URL` to `http://127.0.0.1:2024`
      3. Give it a name like `Local EAIA`
      4. Press `Submit`

You can now interract with EAIA in the Agent Inbox.

## Run in production (LangGraph Cloud)

These instructions will go over how to run EAIA in LangGraph Cloud.
You will need a LangSmith Plus account to be able to access [LangGraph Cloud](https://langchain-ai.github.io/langgraph/concepts/langgraph_cloud/)

If desired, you can always run EAIA in a self-hosted manner using LangGraph Platform [Lite](https://langchain-ai.github.io/langgraph/concepts/self_hosted/#self-hosted-lite) or [Enterprise](https://langchain-ai.github.io/langgraph/concepts/self_hosted/#self-hosted-enterprise).

### Set up EAIA on LangGraph Cloud

1. Make sure you have a LangSmith Plus account
2. Navigate to the deployments page in LangSmith
3. Click `New Deployment`
4. Connect it to your GitHub repo containing this code.
5. Give it a name like `Executive-AI-Assistant`
6. Add the following environment variables
   1. `OPENAI_API_KEY`
   2. `ANTHROPIC_API_KEY`
   3. `GMAIL_SECRET` - This is the value in `eaia/.secrets/secrets.json`
   4. `GMAIL_TOKEN` - This is the value in `eaia/.secrets/token.json`
7. Click `Submit` and watch your EAIA deploy

### Ingest manually

Let's now kick off a manual ingest job to ingest some emails and run them through our LangGraph Cloud EAIA.

First, get your `LANGGRAPH_CLOUD_URL`

To kick off an ingest job, run:

```shell
poetry run python scripts/run_ingest.py --minutes-since 120 --rerun 1 --early 0 --url ${LANGGRAPH-CLOUD-URL}
```

This will ingest all emails in the last 120 minutes (`--minutes-since`). It will NOT break early if it sees an email it already saw (`--early 0`) and it will
rerun ones it has seen before (`--rerun 1`). It will run against the prod instance we have running (`--url ${LANGGRAPH-CLOUD-URL}`)

### Set up Agent Inbox with LangGraph Cloud EAIA

After we have [deployed it](#set-up-eaia-on-langgraph-cloud), we can interract with any results.

1. Go to [Agent Inbox](https://dev.agentinbox.ai/)
2. Connect this to your locally running EAIA agent:
   1. Click into `Settings`
   2. Click `Add Inbox`
      1. Set `Assistant/Graph ID` to `main`
      2. Set `Deployment URL` to your deployment URL
      3. Give it a name like `Prod EAIA`
      4. Press `Submit`

### Set up cron job

You probably don't want to manually run ingest all the time. Using LangGraph Platform, you can easily set up a cron job
that runs on some schedule to check for new emails. You can set this up with:

```shell
poetry run python scripts/setup_cron.py --url ${LANGGRAPH-CLOUD-URL}
```

## Advanced Options

If you want to control more of EAIA besides what the configuration allows, you can modify parts of the code base.

**Reflection Logic**
To control the prompts used for reflection (e.g. to populate memory) you can edit `eaia/reflection_graphs.py`

**Triage Logic**
To control the logic used for triaging emails you can edit `eaia/main/triage.py`

**Calendar Logic**
To control the logic used for looking at available times on the calendar you can edit `eaia/main/find_meeting_time.py`

**Tone & Style Logic**
To control the logic used for the tone and style of emails you can edit `eaia/main/rewrite.py`

**Email Draft Logic**
To control the logic used for drafting emails you can edit `eaia/main/draft_response.py`

## Troubleshooting

### Common Issues

#### 1. Dependency Version Conflicts

**Error**: `langsmith>=0.3.45, we can conclude that eaia==0.1.0 cannot be used`

**Solution**: This happens when deploying to LangGraph Cloud. Update your `pyproject.toml`:
```toml
langsmith = "^0.3.45"  # Instead of "^0.2"
```
Then run:
```bash
poetry update langsmith
git add pyproject.toml poetry.lock
git commit -m "Fix langsmith version for LangGraph deployment"
git push
```

#### 2. Python Version Compatibility

**Error**: `error: the configured Python interpreter version (3.13) is newer than PyO3's maximum supported version (3.12)`

**Solution**: Use Python 3.11.x or set the compatibility flag:
```bash
export PYO3_USE_ABI3_FORWARD_COMPATIBILITY=1
poetry install
```

#### 3. Gmail Authentication Issues

**Error**: `ModuleNotFoundError: No module named 'eaia'`

**Solution**: Use Poetry to run the script:
```bash
poetry run python scripts/setup_gmail.py
```

**Error**: "App has not completed verification"

**Solution**: For personal Gmail accounts:
1. In Google Cloud Console, go to "OAuth consent screen"
2. Select "External" as User Type
3. Add your email as a test user under "Test users"

#### 4. Environment Variables

Make sure to set all required environment variables:
```bash
export OPENAI_API_KEY="your_openai_key"
export ANTHROPIC_API_KEY="your_anthropic_key" 
export LANGSMITH_API_KEY="your_langsmith_key"
```

For LangGraph Cloud deployment, also set:
- `GMAIL_SECRET` - Contents of `eaia/.secrets/secrets.json`
- `GMAIL_TOKEN` - Contents of `eaia/.secrets/token.json`

#### 5. Configuration Issues

**Issue**: Default configuration references LangChain/Harrison

**Solution**: Update `eaia/main/config.yaml` with your personal details:
- Change email addresses to your Gmail account
- Update name, background, timezone
- Customize triage rules for your work/personal context
- Replace company-specific references

### Getting Help

If you encounter other issues:
1. Check the deployment logs in LangGraph Cloud
2. Verify all environment variables are set correctly
3. Ensure your Gmail API credentials have the correct scopes
4. Test locally first before deploying to production
