# FlutterFlow Documentation Assistant

This folder holds all the code: the scraper that loads the FlutterFlow docs into Supabase, and the Streamlit assistant that answers questions from them. See the [root README](../README.md) for the project overview and the full list of limitations.

## Features

- **Ingestion**
  - Reads page URLs from the FlutterFlow docs sitemap
  - Crawls each page with Crawl4AI and converts it to markdown
  - Generates a short summary and a full-page embedding with OpenAI
  - Inserts one row per page into a Supabase `documents` table

- **Documentation search**
  - One agent tool, `search_documentation`, that runs two stages in a fixed order
  - Stage 1 ranks pages by trigram similarity between the question and the page title
  - Stage 2 adds the found titles to the question and runs a vector search
  - Returns both result sets to the model as an overview and a detailed section

- **Streamlit UI**
  - A text input for questions and a "Clear Chat" button
  - Styled user and assistant messages, newest first
  - Conversation memory kept in the running process

## Setup

### Prerequisites

- Python 3.9+
- Supabase account with a project
- OpenAI API key

### Installation

1. Clone the repository:
   ```bash
   git clone https://github.com/S-Sahanii0/flutterflow-doc-agent.git
   cd flutterflow-doc-agent
   ```

2. Install dependencies. The app's packages are in the root file and the scraper's are in this folder:
   ```bash
   pip install -r requirements.txt -r flutterflow_scraper/requirements.txt
   ```
   Crawl4AI drives a headless browser. Follow its own install guide for the browser setup.

3. Create a `.env` file in the repo root (not in this folder):
   ```
   SUPABASE_URL=your_supabase_url
   SUPABASE_KEY=your_supabase_key
   OPENAI_API_KEY=your_openai_api_key
   ```

### Supabase setup

1. Create a new Supabase project.

2. In the SQL editor, run the contents of `supabase/init.sql`. It enables `pgvector` and creates the `documents` table and the `match_documents` function.

3. Then run the contents of `supabase/search_metadata.sql`. It enables `pg_trgm` and creates the `search_doc_metadata` function.

## Usage

Run both commands from this folder.

1. Load the docs into Supabase:
   ```bash
   cd flutterflow_scraper
   python src/scraper.py
   ```
   The scraper only inserts, so running it twice creates duplicate rows.

2. Start the application:
   ```bash
   streamlit run src/app.py
   ```

3. Open `http://localhost:8501` and ask a question about FlutterFlow.

## Architecture

### Components

1. **Scraper** (`src/scraper.py`)
   - Fetches `sitemap.xml` and skips `/tags/`, `/blog/`, and `/troubleshooting/` paths
   - Crawls pages one at a time
   - Summarizes with `chatgpt-4o-latest` and embeds with `text-embedding-3-small`
   - Saves a JSON copy of the results to `output/`

2. **Agent** (`src/agent.py`)
   - `FlutterFlowAgent` wires the LLM (`gpt-4o`), embeddings, vector store, memory, and prompt
   - Uses a LangChain OpenAI functions agent with `ConversationBufferMemory`

3. **Tools** (`src/tools.py`)
   - Defines `search_documentation`, the only tool the agent can call
   - Combines the metadata search and the vector search inside that one tool

4. **Streamlit UI** (`src/app.py`)
   - Caches one agent for the whole app and calls it for each question
   - Shows the full answer when it is ready. Responses are not streamed.

5. **Supabase** (`supabase/`)
   - `documents` table with a 1536-dimension `pgvector` column
   - `match_documents`: cosine similarity search, cutoff 0.7, limit 3
   - `search_doc_metadata`: trigram similarity search on titles, limit 3

### Search process

1. The agent decides whether to call `search_documentation` and what query to send.
2. The tool calls `search_doc_metadata` and collects the top 3 titles.
3. The tool embeds the query plus those titles and calls `match_documents`.
4. Both result sets go back to the model, which writes the answer.

## Known gaps

- The generated summary is saved in the `summary` column, but the metadata search reads `metadata->>'summary'`, which the scraper does not write. Matching is on the title only.
- The page URL is saved in the `url` column, but both search stages look for it inside `metadata`. Results carry the docs home page URL instead.
- Pages are embedded whole, with no chunking.
- No vector or trigram index is created.
- Memory is shared by all browser sessions and is lost on restart.

## Contributing

1. Fork the repository
2. Create a feature branch
3. Commit your changes
4. Push to the branch
5. Create a Pull Request

## License

This project is licensed under the MIT License. See the `LICENSE` file in this folder.
