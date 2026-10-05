# FlutterFlow Documentation Agent

A RAG-style assistant that scrapes the official FlutterFlow documentation, stores it in Supabase (Postgres with pgvector), and answers questions through a Streamlit UI.

Built as a side project to help my team at work and to learn how retrieval-augmented generation works end to end.

<!-- ADD SCREENSHOT OF THE CHAT UI HERE -->

## What it does

- Reads page URLs from the FlutterFlow docs sitemap and crawls each page with Crawl4AI
- Summarizes each page with an OpenAI chat model and embeds the full page with `text-embedding-3-small`
- Stores one row per page in a Supabase `documents` table
- Answers questions with a LangChain agent (GPT-4o) that has one search tool
- Keeps conversation history in memory while the app is running

## Architecture

```
Ingestion:  sitemap.xml -> Crawl4AI -> summary + embedding (OpenAI) -> Supabase documents table
Query:      app.py (Streamlit) -> agent.py (LangChain + GPT-4o) -> tools.py -> Supabase SQL functions
```

| File | Role |
|---|---|
| `flutterflow_scraper/src/scraper.py` | Fetches the sitemap, crawls pages one at a time, summarizes, embeds, and inserts into Supabase. Also writes a JSON copy to `output/`. |
| `flutterflow_scraper/src/tools.py` | Defines the single agent tool, `search_documentation` |
| `flutterflow_scraper/src/agent.py` | Wires the LLM, embeddings, vector store, memory, and prompt into an agent |
| `flutterflow_scraper/src/app.py` | Streamlit UI: a text input and a newest-first message list |
| `flutterflow_scraper/supabase/init.sql` | Enables `pgvector`, creates the `documents` table and the `match_documents` function |
| `flutterflow_scraper/supabase/search_metadata.sql` | Enables `pg_trgm` and creates the `search_doc_metadata` function |

### How a search works

The agent decides whether to call `search_documentation`. Each call runs two stages in a fixed order:

1. Metadata search: `search_doc_metadata` ranks pages by trigram similarity between the question and the page title, and returns the top 3.
2. Vector search: the question plus the titles from stage 1 are embedded and passed to `match_documents`, which returns up to 3 pages with cosine similarity above 0.7.

Both result sets are returned to the model as text, and the model writes the answer.

## Setup

1. Clone the repo and install dependencies. The app and the scraper have separate requirements files:
   ```
   git clone https://github.com/S-Sahanii0/flutterflow-doc-agent.git
   cd flutterflow-doc-agent
   pip install -r requirements.txt -r flutterflow_scraper/requirements.txt
   ```
   Crawl4AI drives a headless browser. Follow its own install guide for the browser setup.
2. Create a `.env` file in the repo root:
   ```
   SUPABASE_URL=your_supabase_url
   SUPABASE_KEY=your_supabase_key
   OPENAI_API_KEY=your_openai_api_key
   ```
3. In the Supabase SQL editor, run the scripts in this order:
   1. `flutterflow_scraper/supabase/init.sql`
   2. `flutterflow_scraper/supabase/search_metadata.sql`

   `init.sql` goes first because it creates the `documents` table that the second script's function queries.
4. Scrape the docs:
   ```
   cd flutterflow_scraper
   python src/scraper.py
   ```
5. Start the assistant from the same folder:
   ```
   streamlit run src/app.py
   ```
   Then open `http://localhost:8501`.

## Tech stack

- Python 3.9+
- LangChain (`langchain`, `langchain-openai`, `langchain-community`)
- OpenAI API: `gpt-4o` for answers, `chatgpt-4o-latest` for page summaries, `text-embedding-3-small` for embeddings
- Supabase (Postgres) with `pgvector` and `pg_trgm`
- Crawl4AI, requests, lxml, tqdm for ingestion
- Streamlit

## Limitations

- The agent has one tool. The two search stages always run together, so the model does not choose between them.
- The metadata stage matches on the title only, and the title is the last segment of the page URL. The generated summary is saved in the `summary` column, but the search function reads `metadata->>'summary'`, which the scraper does not write.
- The page URL is saved in the `url` column, but both search stages look for it inside `metadata`. Results therefore carry the docs home page URL instead of the page URL.
- Each page is embedded whole. There is no chunking or truncation.
- The 0.7 similarity cutoff and the limit of 3 are hard-coded in SQL. The cutoff has not been tuned or evaluated.
- No vector or trigram index is created, so both searches scan the whole table.
- The scraper only inserts. Running it again creates duplicate rows, and there is no update or delete path for changed docs.
- The crawl skip list (`/tags/`, `/blog/`, `/troubleshooting/`) is hard-coded. The scraper does not fetch `robots.txt`.
- Conversation memory is an unbounded in-process buffer. One agent instance is cached for the whole app, so all browser sessions share it and it is lost on restart.
- Responses are not streamed. There are no tests and no answer-quality evaluation.

## Notes

- This project only uses publicly available FlutterFlow documentation. Check FlutterFlow's terms before scraping at scale.
- This is a small side project, not a production system.

## License

MIT. See `flutterflow_scraper/LICENSE`.
