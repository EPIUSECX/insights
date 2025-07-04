# 4. Backend and API

The backend of the Insights application is built entirely on the Frappe Framework. It handles all business logic, data processing, and communication with the database and external data sources. This document details the server-side architecture, key API endpoints, and background processes.

## 4.1 Server-Side Architecture

The backend logic is organized within the `insights` module of the application. Key directories include:

*   **`insights/insights/doctype`**: Contains the definitions of all custom DocTypes, which form the data model of the application. Each DocType's directory includes its Python controller file (`.py`) where server-side hooks and business logic are implemented.
*   **`insights/api`**: This directory contains standalone Python files that define the core API endpoints (`@frappe.whitelist`) used by the frontend. This is a clean way to separate the primary API from the DocType controller logic.
*   **`insights/insights/doctype/insights_data_source_v3/connectors`**: This crucial directory contains the Python classes for each type of data source connector (e.g., `postgres.py`, `duckdb.py`).

## 4.2 Whitelisted API Endpoints (`@frappe.whitelist`)

The following is a list of key whitelisted functions that serve as the primary API for the frontend. These functions are typically located in the `insights/api` directory or within the Python controllers of specific DocTypes.

The primary interaction with the backend is handled by the methods within the `InsightsDataSourcev3` DocType class, rather than a separate API directory.

### Key `InsightsDataSourcev3` Methods

*   **`_get_db_connection()`**: This is the central factory method. Based on the `database_type` field of the data source document, it calls the appropriate connector function (e.g., `get_postgres_connection`, `get_duckdb_connection`) to get a live connection object from the `ibis` library.

*   **`_get_ibis_backend()`**: A wrapper around `_get_db_connection()` that caches the connection object in `frappe.local.insights_db_connections` for the duration of the request. This prevents re-establishing connections for multiple queries within the same API call. It also handles setting session properties for MariaDB connections, such as `time_zone` and `MAX_STATEMENT_TIME`.

*   **`get_table_list()`**: Retrieves the list of available tables from the connected data source. It has special handling for PostgreSQL to correctly list tables from multiple schemas.

*   **`test_connection()`**: A simple method that calls `get_table_list()` to validate that the credentials are correct and the source is reachable. This is used to set the "Active" or "Inactive" status of the data source.

*   **`update_table_list()`**: This method synchronizes the available tables from the remote source with the `Insights Table v3` DocType in Frappe, effectively caching the schema.

*   **`get_ibis_table(table_name)`**: Returns an `ibis` table expression object for a given table name, which is the starting point for building a query.

## 4.3 Background Jobs & Scheduled Tasks

The Insights application implements a robust connection management strategy to ensure that database connections are handled efficiently and reliably. This is managed through hooks that fire before and after each request.

*   **`before_request()`**: This function, triggered at the start of every API request, ensures that `frappe.local.insights_db_connections` exists as an empty dictionary. This local cache holds all active Ibis connections for the duration of the request.

*   **`after_request()`**: At the end of the request, this function iterates through all the cached connections in `frappe.local.insights_db_connections` and calls the `disconnect()` method on each one. This gracefully closes all external database connections and prevents resource leaks.

*   **`db_connections()`**: This is a Python context manager that wraps the `before_request` and `after_request` logic, providing a clean way to manage the connection lifecycle for specific operations.

## 4.4 Example API Interaction: Loading a Chart

1.  **Frontend Request:** The Vue.js frontend makes a request to an endpoint that needs to run a query (e.g., to render a chart).
2.  **Backend Processing:**
    *   The backend code gets the relevant `Insights Data Source v3` document.
    *   It calls the `_get_ibis_backend()` method on this document.
    *   The `_get_ibis_backend()` method checks the `frappe.local.insights_db_connections` cache.
    *   If a connection is not found, it calls `_get_db_connection()`.
    *   `_get_db_connection()` looks at the `database_type` and calls the appropriate function (e.g., `get_postgres_connection`).
    *   The connector function returns a live `ibis` connection object.
    *   This connection is stored in the `frappe.local` cache.
    *   The query is built and executed using this connection.
3.  **After Request Hook:** Once the request is complete and the response is sent, the `after_request` hook fires, finds the connection object in the cache, and calls its `disconnect()` method.
---

## 4.5 Core Utility Modules

While most backend logic is contained within DocType controllers, there are several key utility modules that provide the core data processing and query-building capabilities.

### 4.5.1 Ibis Query Builder (`ibis_utils.py`)

This file is the heart of the entire Insights application. It contains the `IbisQueryBuilder` class, which is responsible for translating the JSON `operations` stored in an `Insights Query v3` document into a valid, executable `ibis` query.

*   **`IbisQueryBuilder` Class:**
    *   The main `build()` method iterates through the JSON operations array.
    *   For each operation type (e.g., `filter`, `join`, `summarize`), it calls a dedicated `apply_*` method.
    *   Each `apply_*` method uses the fluent API of the `ibis` library to add a new clause or transformation to the query expression.

*   **`execute_ibis_query()` Function:**
    *   This function is the final step in the query pipeline. It takes a completed `ibis` query object and executes it.
    *   **Caching:** It implements a robust caching layer. It generates a unique key for each query based on its SQL and the target database. If a valid result is found in the Frappe cache, it is returned instantly, avoiding a costly database roundtrip. Otherwise, the query is executed, and the results are stored in the cache.
    *   **Logging & Error Handling:** It logs the execution time of every query and includes specific error handling for common database issues like timeouts.

*   **Custom Code & Expression Evaluation:**
    *   The utility module provides a powerful `evaluate_expression` method that uses a custom safe evaluator (`exec_with_return`) to allow users to write custom Python expressions for calculated columns and advanced filters.
    *   It also features an `apply_code` method that allows an entire query to be defined by a Python script that returns a pandas DataFrame, enabling limitless data transformation possibilities within a secure, sandboxed environment.