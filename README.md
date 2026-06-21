ResearchMind — Multi-Agent AI Research System

ResearchMind takes a topic and produces a structured, cited research report by running it through a pipeline of four specialized AI components: a search agent, a reader agent, a writer chain, and a critic chain. Each stage's output feeds the next, so the final report is grounded in live web data rather than the model's own (and possibly outdated) knowledge.

Live demo: https://multi-agent-ai-research-system-51fy.onrender.com

How it works

Topic
  │
  ▼
┌─────────────────┐
│  Search Agent    │  LangChain agent + Tavily API
│                  │  Finds recent, relevant URLs and snippets for the topic
└────────┬─────────┘
         ▼
┌─────────────────┐
│  Reader Agent    │  LangChain agent + custom scraping tool
│                  │  Picks the most relevant URL from search results and
│                  │  scrapes it (BeautifulSoup) for deeper content
└────────┬─────────┘
         ▼
┌─────────────────┐
│  Writer Chain    │  Prompt → Gemini 2.5 Flash → StrOutputParser
│                  │  Synthesizes search results + scraped content into a
│                  │  structured report (Introduction, Key Findings,
│                  │  Conclusion, Sources)
└────────┬─────────┘
         ▼
┌─────────────────┐
│  Critic Chain    │  Prompt → Gemini 2.5 Flash → StrOutputParser
│                  │  Independently scores the report (X/10), lists
│                  │  strengths and areas to improve, gives a one-line
│                  │  verdict
└────────┬─────────┘
         ▼
   Report + Critique

The search and reader stages are implemented as LangChain agents (via create_agent) that reason over which tool calls to make, rather than fixed function calls — the search agent decides how to query, and the reader agent decides which scraped URL to pursue based on the search results it receives. The writer and critic stages are deterministic LCEL chains (prompt → LLM → parser), since by that point the task is "transform this text into a structured artifact" rather than "decide what action to take next."

Tech stack

ComponentTechnologyLLMGemini 2.5 Flash (langchain-google-genai)Agent frameworkLangChain (create_agent)Web searchTavily Search APIWeb scrapingRequests + BeautifulSoup4OrchestrationPython, LangChain Expression Language (LCEL)Frontend(your frontend framework here)DeploymentRender

Project structure

.
├── agents.py       # Agent and chain definitions (search agent, reader agent,
│                   # writer chain, critic chain)
├── pipeline.py      # run_research_pipeline() — orchestrates all 4 stages in sequence
├── tools.py          # web_search and scrape_url tool implementations
└── ...               # frontend / app entrypoint

Setup

1. Clone and install dependencies

bashgit clone https://github.com/<Sayed2003-lgtm>/researchmind.git
cd researchmind
pip install -r requirements.txt

2. Configure environment variables

Create a .env file in the project root:

envGOOGLE_API_KEY=your_gemini_api_key
TAVILY_API_KEY=your_tavily_api_key


Get a Gemini API key from Google AI Studio
Get a Tavily API key from tavily.com


3. Run the pipeline

bashpython pipeline.py

You'll be prompted for a research topic, and the pipeline will print progress through each stage (search → scrape → write → critique) along with the final report and critique.

Example output

Each run produces:


Search Results — titles, URLs, and snippets from the top web results for the topic
Scraped Content — deep text extracted from the most relevant source
Research Report — a structured write-up with Introduction, Key Findings (3+ points), Conclusion, and a Sources list
Critic Feedback — a strict X/10 score, strengths, areas to improve, and a one-line verdict


Design notes & known limitations


Sequential, not parallel. The four stages run one after another, with each stage's output feeding the next — this isn't a coordinator agent dynamically routing between sub-agents.
Single-source depth. The reader agent currently scrapes one URL per run for deeper content (the search agent independently surfaces up to 5 results as supporting context for the writer). Scraping multiple top results would likely improve report depth at the cost of latency.
Scrape robustness. The scraper strips script, style, nav, and footer tags and truncates to 3,000 characters; it doesn't handle JavaScript-rendered pages, paywalls, or robots.txt restrictions.
No caching/persistence. Each run re-searches and re-scrapes from scratch; there's no vector store or report history yet.


Possible next steps


Scrape and synthesize multiple sources per run instead of one
Add a retry/self-correction loop where the critic's feedback triggers a writer revision pass
Persist past reports (e.g., in a lightweight DB) so users can revisit prior research
Stream agent progress to the frontend in real time instead of polling for completion