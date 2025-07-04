# 5. Data Sources and Extensibility

The ability to connect to a wide variety of data sources is a core feature of the Insights application. This document details how existing connectors work and provides a step-by-step guide for developers to add new data source providers.

## 5.1 Existing Data Connectors

The Insights application manages data source connections through the `Insights Data Source V3` DocType. The actual logic for connecting to and querying different source types is encapsulated in a set of Python classes located in `insights/insights/doctype/insights_data_source_v3/connectors/`.

Each connector inherits from a base class and implements a standardized interface for:
*   **Connecting:** Establishing a connection using credentials stored in the DocType.
*   **Schema Discovery:** Fetching metadata about the source, such as table names, column names, and data types.
*   **Query Execution:** Translating a generic query definition from `Insights Query V3` into the native language of the source (e.g., SQL) and executing it.

### 5.1.1 PostgreSQL Connector

*   **File:** `insights/insights/doctype/insights_data_source_v3/connectors/postgresql.py`
*   **Function:** `get_postgres_connection(data_source)`

This connector uses the `ibis.postgres.connect()` method to establish a connection. It supports two modes of authentication:
1.  **Connection String:** If a `connection_string` is provided on the Data Source document, it is used directly. The string is URL-encoded for safety.
2.  **Component Fields:** If no connection string is present, it constructs the connection from individual fields: `host`, `port`, `username`, `password`, `database_name`, and `schema`. It also supports an optional `use_ssl` flag.

### 5.1.2 DuckDB Connector

*   **File:** `insights/insights/doctype/insights_data_source_v3/connectors/duckdb.py`
*   **Function:** `get_duckdb_connection(data_source, read_only=True)`

This connector is more versatile and can connect to DuckDB in two ways:
1.  **Remote Database:** If the `database_name` starts with `http`, it uses the `httpfs` extension to connect to a remote DuckDB database file over the network.
2.  **Local File:** Otherwise, it treats the `database_name` as a local file name. It resolves the full path within the site's private files directory (`/private/files/`) and connects to the `.duckdb` file. If the file does not exist, it is created.

---

## 5.2 Developer Guide: Adding a New Data Source

This guide provides a practical, step-by-step tutorial for creating a new data source connector. For this example, we will create a hypothetical connector for a **REST API that returns CSV data**.

### Step 1: Create the Connector Python Class

Create a new file in `insights/insights/doctype/insights_data_source_v3/connectors/` named `rest_csv.py`.

The connector is a simple Python function that takes the `data_source` document as an argument and returns an `ibis` connection object.

Create a new file in `insights/insights/doctype/insights_data_source_v3/connectors/` named `rest_csv.py`.

```python
# insights/insights/doctype/insights_data_source_v3/connectors/rest_csv.py

import frappe
import ibis
import requests
import pandas as pd
from io import StringIO

def get_rest_csv_connection(data_source):
    """
    This function creates a DuckDB in-memory database and populates it
    with data from a CSV-returning REST API. It then returns an Ibis
    connection to this in-memory database.
    """
    api_url = data_source.get_password(fieldname='api_url', raise_exception=True)
    api_key = data_source.get_password(fieldname='api_key', raise_exception=False)
    
    headers = {}
    if api_key:
        headers['Authorization'] = f'Bearer {api_key}'

    try:
        response = requests.get(api_url, headers=headers, timeout=10)
        response.raise_for_status()
    except requests.RequestException as e:
        frappe.throw(f"Failed to fetch data from API: {e}")

    # Use pandas to parse the CSV data
    csv_data = StringIO(response.text)
    df = pd.read_csv(csv_data)

    # Create an in-memory DuckDB database and load the DataFrame
    # Ibis can then query this DataFrame as if it were a table.
    con = ibis.duckdb.connect()
    table_name = frappe.scrub(data_source.title)
    con.create_table(table_name, df)
    
    return con
```

### Step 2: Update the `Insights Data Source V3` DocType

You need to add "REST CSV" as a `Select` option to the `source_type` field in the `insights_data_source_v3.json` file.

```json
// in insights/insights/doctype/insights_data_source_v3/insights_data_source_v3.json
{
    "fieldname": "source_type",
    "fieldtype": "Select",
    "label": "Source Type",
    "options": "PostgreSQL\nDuckDB\nREST CSV", // Add the new option here
    "reqd": 1
},
```

You also need to add fields for the API URL and API Key. Use the `depends_on` property to only show them when "REST CSV" is selected.

```json
{
    "fieldname": "api_url",
    "fieldtype": "Data",
    "label": "API URL",
    "depends_on": "eval:doc.source_type == 'REST CSV'"
},
{
    "fieldname": "api_key",
    "fieldtype": "Password",
    "label": "API Key (Optional)",
    "depends_on": "eval:doc.source_type == 'REST CSV'"
}
```

### Step 3: Update the Connector Factory

In `insights/insights/doctype/insights_data_source_v3/insights_data_source_v3.py`, import your new connector function and add it to the `_get_db_connection` method.

```python
# in insights/insights/doctype/insights_data_source_v3/insights_data_source_v3.py

# ... (other imports)
from .connectors.rest_csv import get_rest_csv_connection # 1. Import the function

class InsightsDataSourcev3(InsightsDataSourceDocument, Document):
    # ...

    def _get_db_connection(self) -> BaseBackend:
        # ... (other conditions)
        if self.database_type == "BigQuery":
            return get_bigquery_connection(self)
        if self.database_type == "REST CSV": # 2. Add the new condition
            return get_rest_csv_connection(self)
        if self.database_type == "MSSQL":
            return get_mssql_connection(self)

        frappe.throw(f"Unsupported database type: {self.database_type}")

```

### Step 4: Test

After running `bench migrate`, you can now create a new `Insights Data Source V3`, select "REST CSV" as the type, provide your API URL, and use it in the Query Builder.