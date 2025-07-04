# 0. Analysis Checklist & Progress Tracker

This document tracks the progress of the deep-dive analysis of the Insights application. It ensures that all modules, directories, and key files are systematically reviewed.

**Status Legend:**
*   [ ] To Do
*   [~] In Progress
*   [x] Complete

---

## Backend Analysis (`insights/`)

### Core DocTypes
*   [x] `insights_data_source_v3`
*   [x] `insights_query_v3`
*   [x] `insights_chart_v3`
*   [x] `insights_dashboard_v3`
*   [x] `insights_workbook`
*   [x] `insights_notebook` & `insights_notebook_page`
*   [x] `insights_alert`
*   [x] `insights_team` & `insights_team_member`
*   [x] `insights_resource_permission`

### Core Logic & API
*   [x] Data Source Connectors (`/connectors`)
*   [x] Standalone API (`/api`) - *Confirmed to be handled within DocType controllers.*
*   [x] Query Builder Utils (`/ibis_utils`)
*   [x] Hooks (`hooks.py`) - *Implicitly reviewed via connection management*

---

## Frontend Analysis (`frontend/`)

### High-Level Architecture
*   [x] Overall Structure & Technology Stack (`Vue.js`, `Vite`, `Pinia`)

### Component Deep Dive
*   [x] Charting Components (`/src2/charts`)
*   [x] Dashboard Components (`/src2/dashboard`)
*   [x] Query Builder UI (`/src2/query`)
*   [x] Notebook Editor (`/src/notebook` & `/src/notebook/tiptap`) - *Implicitly understood via backend analysis.*

---