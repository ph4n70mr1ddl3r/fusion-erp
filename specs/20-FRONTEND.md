# 20 - Frontend / UI Specification

## 1. Overview

The frontend is a single-page application (SPA) built with Rust compiling to WebAssembly (WASM), using the Leptos framework for reactive UI components, styled with Tailwind CSS.

---

## 2. Technology Stack

### 2.1 Core Framework
- **Leptos 0.7+** — Full-stack Rust reactive UI framework
- CSR (Client-Side Rendering) mode for SPA behavior
- Reactive signals for state management
- Component-based architecture

### 2.2 Build Toolchain
- **trunk** — WASM build tool and dev server
- **wasm-bindgen** — Rust/WASM interop
- **web-sys** — Browser API bindings

### 2.3 Styling
- **Tailwind CSS** (CDN loaded in index.html)
- Utility-first CSS classes
- Dark/light theme via Tailwind dark mode
- Responsive design (mobile-first)

---

## 3. Application Structure

```
frontend/
├── index.html
├── Cargo.toml
├── src/
│   ├── main.rs                     # Entry point, mount app
│   ├── app.rs                      # Root component, router
│   ├── state/
│   │   ├── auth.rs                 # Auth state (token, user, tenant)
│   │   ├── app_state.rs            # Global app state
│   │   └── mod.rs
│   ├── routes/
│   │   ├── login.rs
│   │   ├── dashboard.rs
│   │   ├── gl/
│   │   │   ├── accounts.rs
│   │   │   ├── journals.rs
│   │   │   ├── periods.rs
│   │   │   └── mod.rs
│   │   ├── ap/
│   │   │   ├── suppliers.rs
│   │   │   ├── invoices.rs
│   │   │   ├── payments.rs
│   │   │   └── mod.rs
│   │   ├── ar/
│   │   │   ├── customers.rs
│   │   │   ├── invoices.rs
│   │   │   ├── receipts.rs
│   │   │   └── mod.rs
│   │   ├── inv/
│   │   │   ├── items.rs
│   │   │   ├── warehouses.rs
│   │   │   ├── movements.rs
│   │   │   └── mod.rs
│   │   ├── proc/
│   │   │   ├── requisitions.rs
│   │   │   ├── purchase_orders.rs
│   │   │   └── mod.rs
│   │   ├── om/
│   │   │   ├── orders.rs
│   │   │   ├── shipments.rs
│   │   │   └── mod.rs
│   │   ├── settings/
│   │   │   ├── users.rs
│   │   │   ├── roles.rs
│   │   │   └── mod.rs
│   │   └── mod.rs
│   ├── components/
│   │   ├── layout.rs               # Sidebar, Header, PageContainer
│   │   ├── tables.rs               # DataTable component
│   │   ├── forms.rs                # Form components
│   │   ├── charts.rs               # Chart components
│   │   ├── modals.rs               # Modal/Dialog components
│   │   └── mod.rs
│   ├── api/
│   │   ├── client.rs               # HTTP client (reqwasm/gloo)
│   │   ├── auth.rs                 # Auth API calls
│   │   ├── gl.rs                   # GL API calls
│   │   ├── ap.rs                   # AP API calls
│   │   └── mod.rs
│   └── utils/
│       ├── format.rs               # Money, date formatting
│       ├── validation.rs           # Client-side validation
│       └── mod.rs
```

---

## 4. Layout Design

### 4.1 Main Layout
```
┌──────────────────────────────────────────────────────────┐
│ Header: Logo | Search | Notifications | User Menu        │
├────────┬─────────────────────────────────────────────────┤
│        │  Breadcrumb: Home > GL > Journal Entries         │
│  Side  │─────────────────────────────────────────────────│
│  bar   │                                                 │
│        │  Page Content Area                               │
│ - GL   │                                                 │
│ - AP   │  [Action Buttons]                               │
│ - AR   │  [Filter Bar]                                   │
│ - FA   │  [DataTable]                                    │
│ - CM   │    ID | Number | Date | Amount | Status         │
│ - Proc │    ... | ...    | ...  | ...    | ...           │
│ - Inv  │                                                 │
│ - OM   │  [Pagination]                                   │
│ - Mfg  │                                                 │
│ - PM   │                                                 │
│ - Rpt  │                                                 │
│ - Adm  │                                                 │
└────────┴─────────────────────────────────────────────────┘
```

### 4.2 Sidebar Navigation Structure
```
Dashboard
Financials
  ├── General Ledger
  │   ├── Accounts
  │   ├── Journal Entries
  │   ├── Periods
  │   └── Budgets
  ├── Accounts Payable
  │   ├── Suppliers
  │   ├── Invoices
  │   ├── Payments
  │   └── Reports
  ├── Accounts Receivable
  │   ├── Customers
  │   ├── Invoices
  │   ├── Receipts
  │   └── Reports
  ├── Fixed Assets
  └── Cash Management
Supply Chain
  ├── Procurement
  │   ├── Requisitions
  │   ├── Purchase Orders
  │   └── Goods Receipts
  ├── Inventory
  │   ├── Items
  │   ├── Warehouses
  │   ├── Movements
  │   └── Physical Counts
  ├── Order Management
  │   ├── Sales Orders
  │   ├── Shipments
  │   └── Returns
  └── Manufacturing
      ├── BOMs
      ├── Work Orders
      └── Production
Projects
  ├── Projects
  ├── Time Entry
  ├── Expenses
  └── Billing
Reports
  ├── Financial Reports
  ├── Operational Reports
  └── Dashboards
Administration
  ├── Users
  ├── Roles & Permissions
  ├── Workflow Rules
  ├── Tenant Settings
  └── Audit Log
```

---

## 5. Shared Components

### 5.1 DataTable Component
```rust
#[component]
pub fn DataTable<T: 'static>(
    columns: Vec<ColumnDef>,
    data: Signal<Vec<T>>,
    #[prop(optional)] on_sort: Option<Callback<(String, SortDir), ()>>,
    #[prop(optional)] on_page: Option<Callback<Option<String>, ()>>,
    #[prop(optional)] pagination: Option<PaginationInfo>,
    #[prop(optional)] actions: Vec<ActionDef>,
) -> impl IntoView { ... }
```

Features:
- Column sorting (click header)
- Column filtering (per-column dropdown)
- Pagination controls
- Row selection (single/multi)
- Row actions (edit, delete, custom)
- Responsive (scrollable on mobile)

### 5.2 Form Components
- `TextInput` — with label, validation, error display
- `SelectInput` — dropdown with search
- `DatePicker` — date selection
- `MoneyInput` — currency amount with formatting
- `TextArea` — multi-line text
- `Checkbox`, `RadioGroup`, `Toggle`
- `Form` — wrapper with validation and submit handling

### 5.3 Chart Components
- `BarChart` — for comparisons
- `LineChart` — for trends over time
- `PieChart` — for proportions
- `GaugeChart` — for KPIs
- Chart library: chart.js via wasm-bindgen or pure Rust svg rendering

---

## 6. API Client

### 6.1 HTTP Client
```rust
pub struct ApiClient {
    base_url: String,
    token: Signal<Option<String>>,
    tenant_id: Signal<Option<String>>,
}

impl ApiClient {
    pub async fn get<T: DeserializeOwned>(&self, path: &str) -> FusionResult<T> {
        let url = format!("{}{}", self.base_url, path);
        let resp = gloo_net::http::Request::get(&url)
            .header("Authorization", &format!("Bearer {}", self.token.get().unwrap()))
            .header("X-Tenant-Id", self.tenant_id.get().unwrap())
            .send()
            .await?;
        let body: ApiResponse<T> = resp.json().await?;
        Ok(body.data)
    }

    pub async fn post<T: Serialize, R: DeserializeOwned>(&self, path: &str, body: &T) -> FusionResult<R> {
        // Similar implementation
        todo!()
    }
}
```

### 6.2 State Management
```rust
// Global auth state
#[derive(Clone, Copy)]
pub struct AuthState {
    pub token: Signal<Option<String>>,
    pub user: Signal<Option<UserInfo>>,
    pub tenant_id: Signal<Option<String>>,
    pub permissions: Signal<Vec<String>>,
}

impl AuthState {
    pub fn has_permission(&self, permission: &str) -> bool {
        self.permissions.get().contains(&permission.to_string())
    }
}
```

---

## 7. Page Templates

### 7.1 List Page Pattern
Every entity list page follows this structure:
1. Page header with title and "Create" button
2. Filter bar (status, date range, search)
3. DataTable with standard columns
4. Pagination controls

### 7.2 Detail/Edit Page Pattern
1. Breadcrumb navigation
2. Header with document number, status badge, action buttons
3. Tab navigation (Header, Lines, Activity, Attachments)
4. Form sections with labeled fields
5. Line item table (for documents with lines)
6. Totals/summary section

### 7.3 Dashboard Page
1. KPI cards row (Revenue, Expenses, Cash, Open AR)
2. Chart row (Revenue trend, Expense breakdown)
3. Tables row (Recent transactions, Pending approvals)
4. Customizable widget layout

---

## 8. Build & Development

### 8.1 Development Server
```bash
# Install trunk
cargo install trunk

# Run dev server with hot reload
cd frontend
trunk serve --open
```

### 8.2 Production Build
```bash
trunk build --release
# Output: dist/ directory with index.html, WASM, JS glue
```

### 8.3 index.html Template
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Fusion ERP</title>
    <link href="https://cdn.jsdelivr.net/npm/tailwindcss@2.2.19/dist/tailwind.min.css" rel="stylesheet">
    <link data-trunk rel="rust" data-wasm-opt="2" />
</head>
<body>
    <div id="app"></div>
</body>
</html>
```
