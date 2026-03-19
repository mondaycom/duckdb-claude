# duckdb-claude

Claude Code configuration and skills for working with the DuckDB codebase.

## Installation

1. Clone the DuckDB repository (if you haven't already):
   ```bash
   git clone git@github.com:duckdb/duckdb.git
   cd duckdb
   ```

2. Clone this repo into the `.claude` directory:
   ```bash
   git clone git@github.com:mondaycom/duckdb-claude.git .claude
   ```

3. Ensure `.claude` is locally excluded from the parent DuckDB repo (it likely already is):
   ```bash
   echo '.claude/' >> .git/info/exclude
   ```

That's it. Claude Code will automatically pick up the skills and settings from the `.claude` directory.
