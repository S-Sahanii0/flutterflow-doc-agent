# FlutterFlow Documentation Agent

A RAG-style assistant that scrapes the official FlutterFlow documentation and community content, stores it in Supabase, and answers questions through a Streamlit chat interface.

Built as a side project to help my team at work and to learn how retrieval-augmented generation works end to end.

<!-- ADD SCREENSHOT OF THE CHAT UI HERE -->

## What it does

- Scrapes the official FlutterFlow docs and stores them in Supabase (Postgres)
- Searches docs using both metadata and content matching (Postgres `pg_trgm`)
- Combines official documentation with community examples in its answers
- Runs a LangChain agent with OpenAI models behind a Streamlit chat UI

## Architecture

```
scraper.py  ->  Supabase (documents table)  ->  tools.py (search)  ->  agent.py (LangChain + OpenAI)  ->  app.py (Streamlit)
```

| File | Role |
|---|---|
| `flutterflow_scraper/src/scraper.py` | Scrapes the FlutterFlow docs |
| `flutterflow_scraper/src/tools.py` | Search and processing tools |
| `flutterflow_scraper/src/agent.py` | Agent implementation |
| `flutterflow_scraper/src/app.py` | Streamlit chat interface |
| `flutterflow_scraper/supabase/` | SQL setup scripts |

## Setup

1. Clone the repo and install dependencies:
   ```
   git clone https://github.com/S-Sahanii0/flutterflow-doc-agent.git
   cd flutterflow-doc-agent
   pip install -r requirements.txt
   ```
2. Create a `.env` file:
   ```
   SUPABASE_URL=your_supabase_url
   SUPABASE_KEY=your_supabase_key
   OPENAI_API_KEY=your_openai_api_key
   ```
3. In Supabase, run `supabase/search_metadata.sql` (enables `pg_trgm`) and `supabase/init.sql` (creates the documents table).
4. Scrape the docs:
   ```
   cd flutterflow_scraper
   python src/scraper.py
   ```
5. Start the assistant:
   ```
   streamlit run src/app.py
   ```
   Then open `http://localhost:8501`.

## Tech stack

Python 3.8+, LangChain, OpenAI API, Supabase, Streamlit, BeautifulSoup, requests

## Notes

- This project only uses publicly available FlutterFlow documentation. Check FlutterFlow's terms before scraping at scale.
- This is a small side project, not a production system.

## License

MIT. See the LICENSE file.
