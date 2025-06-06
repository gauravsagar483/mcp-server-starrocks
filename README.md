[![MseeP.ai Security Assessment Badge](https://mseep.net/mseep-audited.png)](https://mseep.ai/app/starrocks-mcp-server-starrocks)

# StarRocks Official MCP Server

The StarRocks MCP Server acts as a bridge between AI assistants and StarRocks databases. It allows for direct SQL execution, database exploration, data visualization via charts, and retrieving detailed schema/data overviews without requiring complex client-side setup.

<a href="https://glama.ai/mcp/servers/@StarRocks/mcp-server-starrocks">
  <img width="380" height="200" src="https://glama.ai/mcp/servers/@StarRocks/mcp-server-starrocks/badge" alt="StarRocks Server MCP server" />
</a>

## Features

- **Direct SQL Execution:** Run `SELECT` queries (`read_query`) and DDL/DML commands (`write_query`).
- **Database Exploration:** List databases and tables, retrieve table schemas (`starrocks://` resources).
- **System Information:** Access internal StarRocks metrics and states via the `proc://` resource path.
- **Detailed Overviews:** Get comprehensive summaries of tables (`table_overview`) or entire databases (`db_overview`), including column definitions, row counts, and sample data.
- **Data Visualization:** Execute a query and generate a Plotly chart directly from the results (`query_and_plotly_chart`).
- **Intelligent Caching:** Table and database overviews are cached in memory to speed up repeated requests. Cache can be bypassed when needed.
- **Flexible Configuration:** Set connection details and behavior via environment variables.

## Configuration

The MCP server is typically run via an MCP host. Configuration is passed to the host, specifying how to launch the StarRocks MCP server process.

**Using `uv` with installed package:**

```json
{
  "mcpServers": {
    "mcp-server-starrocks": {
      "command": "uv",
      "args": ["run", "--with", "mcp-server-starrocks", "mcp-server-starrocks"],
      "env": {
        "STARROCKS_HOST": "default localhost",
        "STARROCKS_PORT": "default 9030",
        "STARROCKS_USER": "default root",
        "STARROCKS_PASSWORD": "default empty",
        "STARROCKS_DB": "default empty",
        "STARROCKS_OVERVIEW_LIMIT": "default 20000",
        "STARROCKS_MYSQL_AUTH_PLUGIN":"mysql_clear_password"
      }
    }
  }
}
```

**Using `uv` with local directory (for development):**

```json
{
  "mcpServers": {
    "mcp-server-starrocks": {
      "command": "uv",
      "args": [
        "--directory",
        "path/to/mcp-server-starrocks", // <-- Update this path
        "run",
        "mcp-server-starrocks"
      ],
      "env": {
        "STARROCKS_HOST": "default localhost",
        "STARROCKS_PORT": "default 9030",
        "STARROCKS_USER": "default root",
        "STARROCKS_PASSWORD": "default empty",
        "STARROCKS_DB": "default empty",
        "STARROCKS_OVERVIEW_LIMIT": "default 20000",
        "STARROCKS_MYSQL_AUTH_PLUGIN":"mysql_clear_password"
      }
    }
  }
}
```

**Using Streamable HTTP (recommended for integration):**

```json
{
  "mcpServers": {
    "mcp-server-starrocks": {
      "url": "http://localhost:8000/mcp"
    }
  }
}
```

To start the server in Streamable HTTP mode:

```bash
export MCP_TRANSPORT_MODE=streamable-http
uv run mcp-server-starrocks
```

- The `url` field should point to the Streamable HTTP endpoint of your MCP server (adjust host/port as needed).
- With this configuration, clients can interact with the server using standard JSON over HTTP POST requests. No special SDK is required.
- All tool APIs accept and return standard JSON as described above.

> **Note:**
> The `sse` (Server-Sent Events) mode is deprecated and no longer maintained. Please use Streamable HTTP mode for all new integrations.

**Environment Variables:**

- `STARROCKS_HOST`: (Optional) Hostname or IP address of the StarRocks FE service. Defaults to `localhost`.
- `STARROCKS_PORT`: (Optional) MySQL protocol port of the StarRocks FE service. Defaults to `9030`.
- `STARROCKS_USER`: (Optional) StarRocks username. Defaults to `root`.
- `STARROCKS_PASSWORD`: (Optional) StarRocks password. Defaults to empty string.
- `STARROCKS_DB`: (Optional) Default database to use if not specified in tool arguments or resource URIs. If set, the connection will attempt to `USE` this database. Tools like `table_overview` and `db_overview` will use this if the database part is omitted in their arguments. Defaults to empty (no default database).
- `STARROCKS_OVERVIEW_LIMIT`: (Optional) An _approximate_ character limit for the _total_ text generated by overview tools (`table_overview`, `db_overview`) when fetching data to populate the cache. This helps prevent excessive memory usage for very large schemas or numerous tables. Defaults to `20000`.
- `STARROCKS_MYSQL_AUTH_PLUGIN`: (Optional) Specifies the authentication plugin to use when connecting to the StarRocks FE service. For example, set to `mysql_clear_password` if your StarRocks deployment requires clear text password authentication (such as when using certain LDAP or external authentication setups). Only set this if your environment specifically requires it; otherwise, the default auth_plugin is used.
- `MCP_TRANSPORT_MODE`: (Optional) Communication mode that specifies how the MCP Server exposes its services. Available options:
  - `stdio` (default): Communicates through standard input/output, suitable for MCP Host hosting.
  - `streamable-http` (Streamable HTTP): Starts as a Streamable HTTP Server, supporting RESTful API calls.
  - `sse`: **(Deprecated, not recommended)** Starts in Server-Sent Events (SSE) streaming mode, suitable for scenarios requiring streaming responses. **Note: SSE mode is no longer maintained, it is recommended to use Streamable HTTP mode uniformly.**

## Components

### Tools

- `read_query`

  - **Description:** Execute a SELECT query or other commands that return a ResultSet (e.g., `SHOW`, `DESCRIBE`).
  - **Input:** `{ "query": "SQL query string" }`
  - **Output:** Text content containing the query results in a CSV-like format, including a header row and a row count summary. Returns an error message on failure.

- `write_query`

  - **Description:** Execute a DDL (`CREATE`, `ALTER`, `DROP`), DML (`INSERT`, `UPDATE`, `DELETE`), or other StarRocks command that does not return a ResultSet.
  - **Input:** `{ "query": "SQL command string" }`
  - **Output:** Text content confirming success (e.g., "Query OK, X rows affected") or reporting an error. Changes are committed automatically on success.

- `query_and_plotly_chart`

  - **Description:** Executes a SQL query, loads the results into a Pandas DataFrame, and generates a Plotly chart using a provided Python expression. Designed for visualization in supporting UIs.
  - **Input:**
    ```json
    {
      "query": "SQL query to fetch data",
      "plotly_expr": "Python expression string using 'px' (Plotly Express) and 'df' (DataFrame). Example: 'px.scatter(df, x=\"col1\", y=\"col2\")'"
    }
    ```
  - **Output:** A list containing:
    1.  `TextContent`: A text representation of the DataFrame and a note that the chart is for UI display.
    2.  `ImageContent`: The generated Plotly chart encoded as a base64 PNG image (`image/png`). Returns text error message on failure or if the query yields no data.

- `table_overview`

  - **Description:** Get an overview of a specific table: columns (from `DESCRIBE`), total row count, and sample rows (`LIMIT 3`). Uses an in-memory cache unless `refresh` is true.
  - **Input:**
    ```json
    {
      "table": "Table name, optionally prefixed with database name (e.g., 'db_name.table_name' or 'table_name'). If database is omitted, uses STARROCKS_DB environment variable if set.",
      "refresh": false // Optional, boolean. Set to true to bypass the cache. Defaults to false.
    }
    ```
  - **Output:** Text content containing the formatted overview (columns, row count, sample data) or an error message. Cached results include previous errors if applicable.

- `db_overview`
  - **Description:** Get an overview (columns, row count, sample rows) for _all_ tables within a specified database. Uses the table-level cache for each table unless `refresh` is true.
  - **Input:**
    ```json
    {
      "db": "database_name", // Optional if STARROCKS_DB env var is set.
      "refresh": false // Optional, boolean. Set to true to bypass the cache for all tables in the DB. Defaults to false.
    }
    ```
  - **Output:** Text content containing concatenated overviews for all tables found in the database, separated by headers. Returns an error message if the database cannot be accessed or contains no tables.

### Resources

#### Direct Resources

- `starrocks:///databases`
  - **Description:** Lists all databases accessible to the configured user.
  - **Equivalent Query:** `SHOW DATABASES`
  - **MIME Type:** `text/plain`

#### Resource Templates

- `starrocks:///{db}/{table}/schema`

  - **Description:** Gets the schema definition of a specific table.
  - **Equivalent Query:** `SHOW CREATE TABLE {db}.{table}`
  - **MIME Type:** `text/plain`

- `starrocks:///{db}/tables`

  - **Description:** Lists all tables within a specific database.
  - **Equivalent Query:** `SHOW TABLES FROM {db}`
  - **MIME Type:** `text/plain`

- `proc:///{+path}`
  - **Description:** Accesses StarRocks internal system information, similar to Linux `/proc`. The `path` parameter specifies the desired information node.
  - **Equivalent Query:** `SHOW PROC '/{path}'`
  - **MIME Type:** `text/plain`
  - **Common Paths:**
    - `/frontends` - Information about FE nodes.
    - `/backends` - Information about BE nodes (for non-cloud native deployments).
    - `/compute_nodes` - Information about CN nodes (for cloud native deployments).
    - `/dbs` - Information about databases.
    - `/dbs/<DB_ID>` - Information about a specific database by ID.
    - `/dbs/<DB_ID>/<TABLE_ID>` - Information about a specific table by ID.
    - `/dbs/<DB_ID>/<TABLE_ID>/partitions` - Partition information for a table.
    - `/transactions` - Transaction information grouped by database.
    - `/transactions/<DB_ID>` - Transaction information for a specific database ID.
    - `/transactions/<DB_ID>/running` - Running transactions for a database ID.
    - `/transactions/<DB_ID>/finished` - Finished transactions for a database ID.
    - `/jobs` - Information about asynchronous jobs (Schema Change, Rollup, etc.).
    - `/statistic` - Statistics for each database.
    - `/tasks` - Information about agent tasks.
    - `/cluster_balance` - Load balance status information.
    - `/routine_loads` - Information about Routine Load jobs.
    - `/colocation_group` - Information about Colocation Join groups.
    - `/catalog` - Information about configured catalogs (e.g., Hive, Iceberg).

### Prompts

None defined by this server.

## Caching Behavior

- The `table_overview` and `db_overview` tools utilize an in-memory cache to store the generated overview text.
- The cache key is a tuple of `(database_name, table_name)`.
- When `table_overview` is called, it checks the cache first. If a result exists and the `refresh` parameter is `false` (default), the cached result is returned immediately. Otherwise, it fetches the data from StarRocks, stores it in the cache, and then returns it.
- When `db_overview` is called, it lists all tables in the database and then attempts to retrieve the overview for _each table_ using the same caching logic as `table_overview` (checking cache first, fetching if needed and `refresh` is `false` or cache miss). If `refresh` is `true` for `db_overview`, it forces a refresh for _all_ tables in that database.
- The `STARROCKS_OVERVIEW_LIMIT` environment variable provides a _soft target_ for the maximum length of the overview string generated _per table_ when populating the cache, helping to manage memory usage.
- Cached results, including any error messages encountered during the original fetch, are stored and returned on subsequent cache hits.

## Demo

![MCP Demo Image](mcpserverdemo.jpg)
