# 6. Dependencies and Setup

This document lists the external dependencies required by the Insights application and provides a guide for setting up a local development environment.

## 6.1 Python Dependencies

The Python dependencies are managed by Frappe Bench and are listed in the `requirements.txt` file in the `apps/insights` directory. Key libraries include:

*   **`pandas`**: Used extensively for data manipulation, especially for handling query results and data from various file formats.
*   **`requests`**: Used for making HTTP requests to external APIs in data source connectors.
*   **`ibis-framework`**: A powerful Python library for expressing and executing analytical queries. It is likely used as an abstraction layer over different SQL backends.
*   **`duckdb`**: An in-process analytical database, likely used for high-performance queries on local files (e.g., CSV, Parquet).
*   **`psycopg2-binary`**: The PostgreSQL adapter for Python, used by the PostgreSQL connector.

To install these, you would typically run:
```sh
bench get-app insights <repository_url>
bench setup requirements
```

## 6.2 JavaScript Dependencies

The frontend JavaScript dependencies are managed using `npm` or `yarn` and are defined in the `frontend/package.json` file. Key libraries include:

*   **`vue`**: The core Vue.js library.
*   **`vite`**: The build tool for the frontend application.
*   **`pinia`**: The state management library.
*   **`frappe-ui`**: The standard Frappe UI component library.
*   **`@tiptap/vue-3`**: The core library for the Tiptap rich-text editor used in Notebooks.
*   **`apexcharts`**: A popular charting library, likely used for rendering some of the visualizations.

To install these, navigate to the `frontend` directory and run:
```sh
cd apps/insights/frontend
npm install
```

## 6.3 Development Environment Setup

Follow these steps to set up a clean development environment for the Insights application.

1.  **Set up Frappe Bench:** Ensure you have a working Frappe Bench installation. Refer to the [official Frappe documentation](https://frappeframework.com/docs/user/en/bench) for instructions.

2.  **Create a new site:**
    ```sh
    bench new-site insights.local
    ```

3.  **Get the Insights app:**
    ```sh
    bench get-app <insights_repository_url>
    ```

4.  **Install the app on your site:**
    ```sh
    bench --site insights.local install-app insights
    ```

5.  **Install frontend dependencies:**
    ```sh
    cd apps/insights/frontend
    npm install
    ```

6.  **Start the development servers:**
    *   In your main bench directory, start the Frappe backend server:
        ```sh
        bench start
        ```
    *   In a separate terminal, navigate to `apps/insights/frontend` and start the Vite development server:
        ```sh
        npm run dev
        ```

7.  **Access the application:** You can now access the Insights application at `http://insights.local:8080` (or whichever port Vite assigns). The Vite server will proxy API requests to the Frappe backend.