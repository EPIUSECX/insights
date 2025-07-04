# 2. Core Modules Analysis

This document provides a detailed analysis of the core modules that constitute the Insights application. Each section breaks down the purpose, data model (DocTypes), and key logic for each functional area.

## 2.1 Data Sources (`insights_data_source_v3`)

The Data Source module is the foundation of the Insights application, responsible for managing connections to various internal and external data providers.

*   **Purpose & Functionality:** To securely store connection credentials and metadata for different databases, APIs, or file systems, and to provide a standardized interface for querying them.
*   **Key DocTypes:**
    *   `Insights Data Source V3`: The primary DocType for defining a data source.
*   **Server-Side Logic:**
    *   Connection handling and validation.
    *   Schema caching and retrieval.
*   **Client-Side Logic:**
    *   UI for creating and configuring data sources.

---

## 2.2 Queries (`insights_query_v3`)

The Query module is the analytical core of the Insights application, providing the tools to retrieve and transform data.

### 2.2.1 Purpose & Functionality

The module's primary purpose is to provide a structured and user-friendly way to define, execute, and manage data queries. It abstracts the complexity of the underlying data source and query language, allowing users to build complex data requests through a graphical interface. Queries can also be chained, allowing the results of one query to be used as the input for another.

### 2.2.2 Data Model (`Insights Query V3` DocType)

*   **Schema File:** `insights/insights/doctype/insights_query_v3/insights_query_v3.json`

The data model is remarkably flexible. Instead of having rigid fields for every possible SQL clause, it stores the entire query definition in a single JSON field:
*   **`operations` (JSON):** This field holds an array of objects, where each object represents a step or "operation" in the query pipeline (e.g., select columns, filter data, join with another table, apply aggregations). This structure is what powers the frontend's multi-step query builder.
*   **`linked_queries` (JSON):** This field automatically tracks dependencies when a query uses another query as its source table, enabling robust import/export and dependency management.
*   **`workbook` (Link):** Each query is associated with a parent `Insights Workbook`, providing organizational structure.

### 2.2.3 Server-Side Logic (`insights_query_v3.py`)

The Python controller is the engine that brings the query definition to life.

*   **`IbisQueryBuilder`:** This is the key helper class. Its `build()` method iterates through the `operations` JSON array and programmatically constructs a query expression using the `ibis` library. This is the core translation layer between the user's intent (the JSON) and the executable query.

*   **`execute()`:** This whitelisted method is the main entry point for running a query. It performs the following steps:
    1.  Calls the `build()` method to get the `ibis` query object.
    2.  Passes the query object to a centralized `execute_ibis_query` helper function.
    3.  The helper function executes the query against the data source, handles caching, and times the execution.
    4.  Returns a dictionary containing the generated SQL, the column definitions, the result rows, and the time taken.

*   **Whitelisted Helper Methods:** The controller also provides several other powerful, whitelisted API endpoints for the frontend:
    *   `get_count()`: Efficiently retrieves the total row count for a query.
    *   `download_results()`: Allows downloading the full query result set as a CSV file.
    *   `get_distinct_column_values()`: Fetches unique values for a given column, used to populate filter dropdowns in the UI.

### 2.2.4 Client-Side Logic

*   The frontend features a sophisticated multi-step Query Builder UI.
*   This UI allows users to add, remove, and reorder "operations" (filters, joins, etc.).
*   The state of the builder is serialized into the `operations` JSON format, which is then saved to the `Insights Query V3` document.

---

## 2.3 Charts (`insights_chart_v3`)

The Chart module is responsible for the visualization of data retrieved from the Query module.

### 2.3.1 Purpose & Functionality

This module allows users to select a query and render its results as a visual chart (e.g., Bar, Line, Pie). The configuration is highly flexible, managed by the frontend and stored as a single JSON object on the backend.

### 2.3.2 Data Model (`Insights Chart V3` DocType)

*   **Schema File:** `insights/insights/doctype/insights_chart_v3/insights_chart_v3.json`

The chart document is primarily a pointer to a query, with added visual metadata.
*   **`query` (Link):** A direct link to the `Insights Query v3` document that provides the data for the chart.
*   **`chart_type` (Data):** A string field (e.g., "Bar", "Line", "Donut") that tells the frontend which rendering component to use.
*   **`config` (JSON):** This field stores a complex JSON object that defines the entire visual configuration of the chart, such as which columns to use for the X and Y axes, color schemes, labels, and other options. This configuration is built and managed by the frontend chart builder UI.
*   **`workbook` (Link):** Like queries, charts are organized into a parent `Insights Workbook`.

### 2.3.3 Server-Side Logic (`insights_chart_v3.py`)

The server-side logic for charts is minimal and focused on lifecycle and portability rather than data processing.
*   **`export()`:** When a chart is exported, this method intelligently packages the chart's own data along with a full export of its dependent `Insights Query v3` document. This ensures the exported chart is self-contained and can be imported elsewhere.
*   **`import_chart()`:** This function handles the import of a chart, and crucially, it also imports the dependent query and correctly re-links the two within the target workbook.

---

## 2.4 Dashboards (`insights_dashboard_v3`)

The Dashboard module allows users to arrange multiple charts and filters into a single, shareable view.

### 2.4.1 Purpose & Functionality

Dashboards serve as the primary presentation layer for insights, combining multiple visualizations into a cohesive and interactive grid. They can be shared with other users and even made public.

### 2.4.2 Data Model (`Insights Dashboard V3` DocType)

*   **Schema File:** `insights/insights/doctype/insights_dashboard_v3/insights_dashboard_v3.json`

The dashboard's structure is defined by a single, powerful JSON field.
*   **`items` (JSON):** This field stores an array of objects representing the items on the dashboard. Each object contains its type (e.g., "chart", "filter"), a link to the corresponding document (e.g., the name of the `Insights Chart V3`), and its layout information (x/y position, width/height). This provides the frontend with all the information needed to render the dashboard grid.
*   **`linked_charts` (Table MultiSelect):** This field is automatically populated by the `set_linked_charts` method on the backend. It provides a relational link to all charts used in the `items` array, making it easy to see where a chart is being used.
*   **`preview_image` (Data):** Stores the URL to a generated preview image of the dashboard.

### 2.4.3 Server-Side Logic (`insights_dashboard_v3.py`)

The dashboard controller contains sophisticated logic for features beyond simple data display.
*   **Preview Generation:** The `generate_dashboard_preview()` method orchestrates the creation of a visual preview of the dashboard. It does this by:
    1.  Calling an external microservice (`https://preview.frappe.cloud`).
    2.  Passing the public URL of the dashboard to the service.
    3.  Receiving an image in response.
    4.  Saving this image as a private `File` document in Frappe.
    5.  This entire process is queued as a background job (`frappe.enqueue_doc`) so it doesn't slow down the user's save operation.
*   **Access Control:** The `update_access()` method provides a whitelisted endpoint for managing who can view the dashboard. It integrates directly with Frappe's standard sharing system (`DocShare`), allowing for sharing with individual users or making the dashboard available to everyone in the organization.

---

## 2.5 Workbooks & Notebooks (`insights_workbook`, `insights_notebook`)

Workbooks are the top-level organizational unit in the Insights application.

### 2.5.1 Purpose & Functionality

A Workbook acts as a folder or a project space. It doesn't contain any analytical logic itself but serves as the parent container for a collection of related Queries, Charts, and Dashboards. This provides a clean and intuitive way for users to organize their work.

### 2.5.2 Data Model (`Insights Workbook` DocType)

*   **Schema File:** `insights/insights/doctype/insights_workbook/insights_workbook.json`

The DocType is extremely simple, reflecting its role as a container.
*   **`title` (Data):** The name of the workbook.
*   **Links:** The DocType is linked from `Insights Query v3`, `Insights Chart v3`, and `Insights Dashboard v3`, establishing the parent-child relationship.

### 2.5.3 Server-Side Logic

The `Insights Workbook` has no dedicated Python controller file (`insights_workbook.py`). All of its behavior is the standard, out-of-the-box functionality provided by the Frappe Framework for a basic DocType.

---

## 2.6 Notebooks (`insights_notebook`, `insights_notebook_page`)

The Notebook feature provides a rich, document-centric canvas for data analysis, similar to tools like Notion or Jupyter Notebooks. This functionality is architecturally distinct from the rest of the application, being heavily reliant on the frontend.

### 2.6.1 Purpose & Functionality

Notebooks allow users to create free-form documents that combine formatted text, images, and live-embedded analytics components like Queries and Charts. This enables narrative-driven analysis and data storytelling.

### 2.6.2 Data Model

The feature is split across two DocTypes:

*   **`Insights Notebook`**: A simple container for pages. Its only server-side logic is to prevent the deletion of a default "Uncategorized" notebook.

*   **`Insights Notebook Page`**: This is the core DocType. It represents a single document.
    *   **Schema File:** `insights/insights/doctype/insights_notebook_page/insights_notebook_page.json`
    *   **`content` (Code - JSON):** This single field is the most critical part of the data model. It stores the entire content of the page as a structured JSON object. This object is generated and managed by the frontend's rich-text editor and contains all the text, formatting, and references to embedded charts or queries.

### 2.6.3 Server-Side Logic

*   **Controller File:** `insights/insights/doctype/insights_notebook_page/insights_notebook_page.py`

The Python controller for `Insights Notebook Page` is empty (`pass`). This confirms that the backend's role is purely to save and retrieve the JSON content blob. All of the complex logic for rendering, editing, and interacting with the notebook page resides entirely within the client-side Vue.js application.

### 2.6.4 Client-Side Logic

*   The frontend uses a sophisticated block-based, rich-text editor (likely Tiptap.js, given the file structure) to manage the `content` JSON.
*   This editor allows users to add different "blocks" (paragraphs, headings, code blocks, embedded charts).
*   When a chart or query is embedded, the editor stores a reference to that DocType within the JSON structure.
*   When the page is rendered, the frontend parses the JSON and dynamically loads the appropriate Vue components for each block, including fetching data for any embedded analytics.

---

## 2.7 Alerts (`insights_alert`)

The Alerts module provides a proactive monitoring and notification system based on query results.

### 2.7.1 Purpose & Functionality

Alerts allow users to define conditions based on an `Insights Query v3` and receive notifications via Email or Telegram when those conditions are met. This is used for KPI monitoring, anomaly detection, and data-driven workflow automation.

### 2.7.2 Data Model (`Insights Alert` DocType)

*   **Schema File:** `insights/insights/doctype/insights_alert/insights_alert.json`

The alert document captures the what, when, and how of the notification.
*   **`query` (Link):** A reference to the `Insights Query v3` that the alert is based on.
*   **`condition` (Code):** A Python expression that is evaluated against the query's results. If the expression returns `True` (i.e., the query returns any rows after the condition is applied as a filter), the alert is triggered.
*   **`frequency` (Select):** Defines how often the alert condition is checked (e.g., Hourly, Daily, Weekly). It also supports a custom `cron_format`.
*   **`channel` (Select):** The delivery channel for the notification (Email or Telegram).
*   **`recipients` (Small Text):** A comma-separated list of email addresses.
*   **`message` (Markdown Editor):** A template for the notification message. It can include Jinja templating to display data from the query results.

### 2.7.3 Server-Side Logic (`insights_alert.py`)

The controller contains the full lifecycle logic for the alerting system.

*   **Scheduling (`is_event_due`, `send_alerts`):** The `send_alerts` function is the main entry point, intended to be run by the Frappe scheduler. It iterates through all active alerts and uses the `croniter` library to check if an alert's `frequency` or `cron_format` makes it due for execution.

*   **Condition Evaluation (`evaluate_condition`):** When an alert is due, this method takes the `condition` code and passes it to a special method on the linked `Insights Query v3` document (`evaluate_alert_expression`). This effectively applies the condition as a filter to the query and checks if it produces any results.

*   **Message Rendering (`evaluate_message`):** If the condition is met, this method executes the full query to get the data. It then uses `frappe.render_template` to inject the query results into the `message` markdown template. It provides the results in two formats: `rows` (a list of dictionaries) and `datatable` (a pre-rendered HTML table), giving the user flexibility in the notification design.

*   **Delivery (`send_email_alert`, `send_telegram_alert`):** Based on the selected `channel`, the rendered message is dispatched using either `frappe.sendmail` or the `python-telegram-bot` library. The Telegram API token is configured globally in `Insights Settings`.
---

## 2.8 Permissions & Teams (`insights_team`)

The Insights application implements a sophisticated, team-based permission layer on top of Frappe's standard Role-based permissions. This allows for granular control over who can access which data sources and tables.

### 2.8.1 Purpose & Functionality

The Teams module allows administrators to group users into teams and assign specific data resources to them. This provides a flexible and scalable way to manage data access in a large organization, including powerful row-level security capabilities.

### 2.8.2 Data Model

The permission model is defined by three interconnected DocTypes:

*   **`Insights Team`**: The central DocType for managing permissions. It contains:
    *   `team_members`: A child table linking to `User` documents.
    *   `team_permissions`: A child table linking to `Insights Resource Permission` documents.

*   **`Insights Team Member`**: A simple child table that holds a link to a `User`.

*   **`Insights Resource Permission`**: A child table that defines a single permission rule.
    *   `resource_type` (Link): The type of resource (e.g., "Insights Data Source v3", "Insights Table v3").
    *   `resource_name` (Dynamic Link): A link to the specific document being granted access to.
    *   `table_restrictions` (Data): A field to define row-level security rules as Python expressions.

### 2.8.3 Server-Side Logic (`insights_team.py`)

The controller for `Insights Team` contains the core logic for the entire permission system.

*   **Permission Resolution (`get_allowed_resources_for_user`):** This is the primary function for checking access. It determines which resources a user can see by aggregating the permissions from all the teams they belong to. It also handles implicit permissions (granting access to a Data Source implicitly grants access to all its tables). The results of this function are cached to ensure high performance.

*   **Row-Level Security (`apply_table_restrictions`):** This is the most powerful feature of the permission system. When a query is being built, this function is called. It checks if the user's teams have any `table_restrictions` defined for the table in the query. If they do, it dynamically injects these restrictions as `filter` clauses into the Ibis query object before it's executed. This ensures that users can only see the rows they are authorized to see, and the logic is enforced at the lowest level.

*   **Admin Team:** The system includes special logic for a team named "Admin". Users in this team are automatically assigned the "Insights Admin" role, providing a simple way to manage top-level administrators.