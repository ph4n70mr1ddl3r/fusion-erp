# 201 - Restaurants Service Specification

## 1. Domain Overview

Restaurants Service provides comprehensive restaurant operations management supporting multi-location management with concept types (quick service, casual, fine dining, ghost kitchen), full menu item lifecycle management with allergen tracking, nutritional information, food cost analysis, and seasonal availability, recipe management with ingredient-level costing, preparation steps, quality standards, and versioning, shift scheduling with role-based assignments, labor budgeting, and publish/finalize workflow, and daily operations summaries with sales, labor cost, food cost, waste, customer satisfaction, and speed-of-service KPIs. Enables franchise and multi-region operations with centralized menu governance and local execution flexibility. Integrates with Inventory for ingredient stock management, Time & Labor for employee scheduling, Financials for daily revenue posting, HR for workforce management, and Loyalty for customer rewards.

**Bounded Context:** Restaurant Operations, Menu Management & Multi-Location Management
**Service Name:** `restaurants-service`
**Database:** `data/restaurants.db`
**HTTP Port:** 8219 | **gRPC Port:** 9219

---

## 2. Database Schema

### 2.1 Restaurant Locations
```sql
CREATE TABLE rest_locations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    location_code TEXT NOT NULL,
    location_name TEXT NOT NULL,
    address TEXT NOT NULL,                            -- JSON: {line1, line2, city, state, postal_code, country, latitude, longitude}
    concept_type TEXT NOT NULL
        CHECK(concept_type IN ('QUICK_SERVICE','CASUAL','FINE_DINING','GHOST_KITCHEN')),
    phone TEXT,
    email TEXT,
    operating_hours TEXT NOT NULL,                    -- JSON: {mon: {open, close}, tue: {...}, ...}
    seating_capacity INTEGER,
    manager_id TEXT NOT NULL,
    region TEXT NOT NULL,
    franchise_flag INTEGER NOT NULL DEFAULT 0,
    opened_date TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','CLOSED','RENOVATING')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, location_code)
);

CREATE INDEX idx_rest_locations_tenant_region ON rest_locations(tenant_id, region);
CREATE INDEX idx_rest_locations_tenant_concept ON rest_locations(tenant_id, concept_type);
CREATE INDEX idx_rest_locations_tenant_status ON rest_locations(tenant_id, status);
CREATE INDEX idx_rest_locations_tenant_manager ON rest_locations(tenant_id, manager_id);
CREATE INDEX idx_rest_locations_tenant_active ON rest_locations(tenant_id, is_active);
```

### 2.2 Menu Items
```sql
CREATE TABLE rest_menu_items (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    menu_item_code TEXT NOT NULL,
    name TEXT NOT NULL,
    category TEXT NOT NULL,
    sub_category TEXT,
    description TEXT,
    ingredients TEXT,                                  -- JSON: array of ingredient descriptors
    allergens TEXT,                                    -- JSON: array of allergen codes
    nutritional_info TEXT,                             -- JSON: {calories, fat_g, protein_g, carbs_g, sodium_mg}
    food_cost_cents INTEGER NOT NULL DEFAULT 0,
    selling_price_cents INTEGER NOT NULL DEFAULT 0,
    food_cost_pct REAL NOT NULL DEFAULT 0.0
        CHECK(food_cost_pct >= 0 AND food_cost_pct <= 100),
    preparation_time_minutes INTEGER NOT NULL DEFAULT 0,
    pos_code TEXT,
    image_url TEXT,
    season TEXT NOT NULL DEFAULT 'AVAILABLE'
        CHECK(season IN ('AVAILABLE','SEASONAL','DISCONTINUED')),
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','DRAFT')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, menu_item_code)
);

CREATE INDEX idx_rest_menu_tenant_category ON rest_menu_items(tenant_id, category);
CREATE INDEX idx_rest_menu_tenant_status ON rest_menu_items(tenant_id, status);
CREATE INDEX idx_rest_menu_tenant_season ON rest_menu_items(tenant_id, season);
CREATE INDEX idx_rest_menu_tenant_pos ON rest_menu_items(tenant_id, pos_code);
CREATE INDEX idx_rest_menu_tenant_active ON rest_menu_items(tenant_id, is_active);
```

### 2.3 Recipes
```sql
CREATE TABLE rest_recipes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    recipe_code TEXT NOT NULL,
    recipe_name TEXT NOT NULL,
    ingredients TEXT NOT NULL,                         -- JSON: [{ingredient_id, quantity, unit, cost_cents}]
    preparation_steps TEXT NOT NULL,                   -- JSON: ordered array of step descriptions
    yield_servings INTEGER NOT NULL DEFAULT 1,
    food_cost_per_serving_cents INTEGER NOT NULL DEFAULT 0,
    labor_time_minutes INTEGER NOT NULL DEFAULT 0,
    quality_standards TEXT,                            -- JSON: temperature, presentation, taste criteria
    version INTEGER NOT NULL DEFAULT 1,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, recipe_code)
);

CREATE INDEX idx_rest_recipes_tenant_status ON rest_recipes(tenant_id, status);
CREATE INDEX idx_rest_recipes_tenant_version ON rest_recipes(tenant_id, version);
CREATE INDEX idx_rest_recipes_tenant_active ON rest_recipes(tenant_id, is_active);
```

### 2.4 Shift Schedules
```sql
CREATE TABLE rest_shift_schedules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    location_id TEXT NOT NULL,
    schedule_date TEXT NOT NULL,
    shifts TEXT NOT NULL,                              -- JSON: [{position, employee_id, start_time, end_time, break_minutes, role}]
    total_planned_hours REAL NOT NULL DEFAULT 0.0,
    total_actual_hours REAL NOT NULL DEFAULT 0.0,
    labor_budget_cents INTEGER NOT NULL DEFAULT 0,
    actual_labor_cents INTEGER NOT NULL DEFAULT 0,
    published_flag INTEGER NOT NULL DEFAULT 0,
    published_at TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','PUBLISHED','FINALIZED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (location_id) REFERENCES rest_locations(id),
    UNIQUE(tenant_id, location_id, schedule_date)
);

CREATE INDEX idx_rest_shifts_tenant_location ON rest_shift_schedules(tenant_id, location_id);
CREATE INDEX idx_rest_shifts_tenant_date ON rest_shift_schedules(tenant_id, schedule_date);
CREATE INDEX idx_rest_shifts_tenant_status ON rest_shift_schedules(tenant_id, status);
CREATE INDEX idx_rest_shifts_tenant_published ON rest_shift_schedules(tenant_id, published_flag);
CREATE INDEX idx_rest_shifts_tenant_active ON rest_shift_schedules(tenant_id, is_active);
```

### 2.5 Daily Summaries
```sql
CREATE TABLE rest_daily_summaries (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    location_id TEXT NOT NULL,
    summary_date TEXT NOT NULL,
    sales_cents INTEGER NOT NULL DEFAULT 0,
    guest_count INTEGER NOT NULL DEFAULT 0,
    avg_ticket_cents INTEGER NOT NULL DEFAULT 0,
    labor_cost_cents INTEGER NOT NULL DEFAULT 0,
    labor_cost_pct REAL NOT NULL DEFAULT 0.0,
    food_cost_cents INTEGER NOT NULL DEFAULT 0,
    food_cost_pct REAL NOT NULL DEFAULT 0.0,
    waste_cents INTEGER NOT NULL DEFAULT 0,
    waste_pct REAL NOT NULL DEFAULT 0.0,
    customer_satisfaction_score REAL,
    speed_of_service_seconds INTEGER,
    top_selling_items TEXT,                           -- JSON: array of top item summaries
    summary_status TEXT NOT NULL DEFAULT 'PRELIMINARY'
        CHECK(summary_status IN ('PRELIMINARY','FINAL')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (location_id) REFERENCES rest_locations(id),
    UNIQUE(tenant_id, location_id, summary_date)
);

CREATE INDEX idx_rest_daily_tenant_location ON rest_daily_summaries(tenant_id, location_id);
CREATE INDEX idx_rest_daily_tenant_date ON rest_daily_summaries(tenant_id, summary_date);
CREATE INDEX idx_rest_daily_tenant_status ON rest_daily_summaries(tenant_id, summary_status);
CREATE INDEX idx_rest_daily_tenant_active ON rest_daily_summaries(tenant_id, is_active);
```

---

## 3. REST API Endpoints

### 3.1 Locations
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/restaurants/locations` | List restaurant locations |
| POST | `/api/v1/restaurants/locations` | Create restaurant location |
| GET | `/api/v1/restaurants/locations/{id}` | Get location details |
| PUT | `/api/v1/restaurants/locations/{id}` | Update location |
| PATCH | `/api/v1/restaurants/locations/{id}/status` | Update location status |
| GET | `/api/v1/restaurants/locations/search` | Search locations by region/concept |

### 3.2 Menu Management
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/restaurants/menu-items` | List menu items |
| POST | `/api/v1/restaurants/menu-items` | Create menu item |
| GET | `/api/v1/restaurants/menu-items/{id}` | Get menu item details |
| PUT | `/api/v1/restaurants/menu-items/{id}` | Update menu item |
| PATCH | `/api/v1/restaurants/menu-items/{id}/status` | Update menu item status |
| GET | `/api/v1/restaurants/menu-items/by-category` | List items by category |
| POST | `/api/v1/restaurants/menu-items/{id}/cost-analysis` | Calculate food cost percentage |

### 3.3 Recipes
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/restaurants/recipes` | List recipes |
| POST | `/api/v1/restaurants/recipes` | Create recipe |
| GET | `/api/v1/restaurants/recipes/{id}` | Get recipe details |
| PUT | `/api/v1/restaurants/recipes/{id}` | Update recipe |
| POST | `/api/v1/restaurants/recipes/{id}/new-version` | Create new recipe version |
| GET | `/api/v1/restaurants/recipes/{id}/cost-breakdown` | Get ingredient cost breakdown |

### 3.4 Scheduling
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/restaurants/shift-schedules` | List shift schedules |
| POST | `/api/v1/restaurants/shift-schedules` | Create shift schedule |
| GET | `/api/v1/restaurants/shift-schedules/{id}` | Get schedule details |
| PUT | `/api/v1/restaurants/shift-schedules/{id}` | Update schedule |
| POST | `/api/v1/restaurants/shift-schedules/{id}/publish` | Publish schedule |
| POST | `/api/v1/restaurants/shift-schedules/{id}/finalize` | Finalize schedule with actuals |
| GET | `/api/v1/restaurants/shift-schedules/by-location` | List schedules by location and date range |

### 3.5 Daily Operations
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/restaurants/daily-summaries` | List daily summaries |
| POST | `/api/v1/restaurants/daily-summaries` | Create daily summary |
| GET | `/api/v1/restaurants/daily-summaries/{id}` | Get summary details |
| PUT | `/api/v1/restaurants/daily-summaries/{id}` | Update summary |
| POST | `/api/v1/restaurants/daily-summaries/{id}/finalize` | Finalize daily summary |
| GET | `/api/v1/restaurants/daily-summaries/comparison` | Compare summaries across locations/dates |
| GET | `/api/v1/restaurants/daily-summaries/kpi-dashboard` | Get KPI dashboard data |

---

## 4. Business Rules

### 4.1 Location Management
1. Each location MUST have a unique location_code within a tenant.
2. Location status transitions MUST follow: ACTIVE -> CLOSED; ACTIVE -> RENOVATING -> ACTIVE.
3. A location in CLOSED status MUST NOT accept new shift schedules or daily summaries.
4. operating_hours JSON MUST contain entries for all seven days of the week.
5. Franchise locations (franchise_flag = 1) MUST have a designated region and manager_id.

### 4.2 Menu Item Management
6. menu_item_code MUST be unique within a tenant.
7. food_cost_pct MUST be calculated as (food_cost_cents / selling_price_cents) * 100 when selling_price_cents > 0.
8. selling_price_cents MUST be a positive integer for items with status ACTIVE.
9. allergens JSON MUST be populated before a menu item can transition from DRAFT to ACTIVE.
10. Items with season SEASONAL MUST have defined availability windows; DISCONTINUED items MUST NOT appear on active menus.

### 4.3 Recipe Management
11. recipe_code MUST be unique within a tenant.
12. ingredients JSON MUST contain at least one ingredient with ingredient_id, quantity, unit, and cost_cents.
13. food_cost_per_serving_cents MUST be recalculated whenever ingredients or yield_servings change.
14. New recipe versions MUST increment the version field; previous versions remain accessible via version number.
15. quality_standards JSON SHOULD include temperature, presentation, and taste criteria for FINE_DINING concepts.

### 4.4 Shift Scheduling
16. The combination of (tenant_id, location_id, schedule_date) MUST be unique for shift schedules.
17. Schedule status transitions MUST follow: DRAFT -> PUBLISHED -> FINALIZED; reversal is not permitted.
18. Each shift entry in shifts JSON MUST have position, employee_id, start_time, end_time, and role fields.
19. total_planned_hours MUST equal the sum of all shift durations in the shifts JSON.
20. labor_budget_cents MUST NOT be negative; actual_labor_cents is populated only during FINALIZED status.

### 4.5 Daily Summaries
21. The combination of (tenant_id, location_id, summary_date) MUST be unique.
22. avg_ticket_cents MUST equal sales_cents / guest_count when guest_count > 0.
23. labor_cost_pct MUST be calculated as (labor_cost_cents / sales_cents) * 100 when sales_cents > 0.
24. food_cost_pct MUST be calculated as (food_cost_cents / sales_cents) * 100 when sales_cents > 0.
25. Summary status transitions MUST follow: PRELIMINARY -> FINAL; finalized summaries MUST NOT be modified.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.restaurants.v1;

service RestaurantsService {
    // Location management
    rpc CreateLocation(CreateLocationRequest) returns (CreateLocationResponse);
    rpc GetLocation(GetLocationRequest) returns (GetLocationResponse);
    rpc ListLocations(ListLocationsRequest) returns (ListLocationsResponse);
    rpc UpdateLocationStatus(UpdateLocationStatusRequest) returns (UpdateLocationStatusResponse);

    // Menu management
    rpc CreateMenuItem(CreateMenuItemRequest) returns (CreateMenuItemResponse);
    rpc GetMenuItem(GetMenuItemRequest) returns (GetMenuItemResponse);
    rpc ListMenuItems(ListMenuItemsRequest) returns (ListMenuItemsResponse);
    rpc UpdateMenuItemStatus(UpdateMenuItemStatusRequest) returns (UpdateMenuItemStatusResponse);

    // Recipes
    rpc CreateRecipe(CreateRecipeRequest) returns (CreateRecipeResponse);
    rpc GetRecipe(GetRecipeRequest) returns (GetRecipeResponse);
    rpc ListRecipes(ListRecipesRequest) returns (ListRecipesResponse);
    rpc CreateRecipeVersion(CreateRecipeVersionRequest) returns (CreateRecipeVersionResponse);

    // Shift scheduling
    rpc CreateShiftSchedule(CreateShiftScheduleRequest) returns (CreateShiftScheduleResponse);
    rpc GetShiftSchedule(GetShiftScheduleRequest) returns (GetShiftScheduleResponse);
    rpc PublishSchedule(PublishScheduleRequest) returns (PublishScheduleResponse);
    rpc FinalizeSchedule(FinalizeScheduleRequest) returns (FinalizeScheduleResponse);

    // Daily operations
    rpc CreateDailySummary(CreateDailySummaryRequest) returns (CreateDailySummaryResponse);
    rpc GetDailySummary(GetDailySummaryRequest) returns (GetDailySummaryResponse);
    rpc FinalizeDailySummary(FinalizeDailySummaryRequest) returns (FinalizeDailySummaryResponse);
    rpc GetKPIDashboard(GetKPIDashboardRequest) returns (GetKPIDashboardResponse);
}

message CreateLocationRequest {
    string tenant_id = 1;
    string location_code = 2;
    string location_name = 3;
    string address = 4;
    string concept_type = 5;
    string phone = 6;
    string email = 7;
    string operating_hours = 8;
    int32 seating_capacity = 9;
    string manager_id = 10;
    string region = 11;
    bool franchise_flag = 12;
    string opened_date = 13;
}

message CreateLocationResponse {
    string location_id = 1;
    string location_code = 2;
    string location_name = 3;
    string status = 4;
}

message GetLocationRequest {
    string tenant_id = 1;
    string location_id = 2;
}

message GetLocationResponse {
    string location_id = 1;
    string location_code = 2;
    string location_name = 3;
    string concept_type = 4;
    string region = 5;
    string status = 6;
    string operating_hours = 7;
    int32 seating_capacity = 8;
}

message ListLocationsRequest {
    string tenant_id = 1;
    string region = 2;
    string concept_type = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListLocationsResponse {
    repeated GetLocationResponse locations = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateLocationStatusRequest {
    string tenant_id = 1;
    string location_id = 2;
    string status = 3;
}

message UpdateLocationStatusResponse {
    string location_id = 1;
    string status = 2;
}

message CreateMenuItemRequest {
    string tenant_id = 1;
    string menu_item_code = 2;
    string name = 3;
    string category = 4;
    string sub_category = 5;
    string description = 6;
    string ingredients = 7;
    string allergens = 8;
    string nutritional_info = 9;
    int64 food_cost_cents = 10;
    int64 selling_price_cents = 11;
    int32 preparation_time_minutes = 12;
    string pos_code = 13;
    string image_url = 14;
    string season = 15;
}

message CreateMenuItemResponse {
    string menu_item_id = 1;
    string menu_item_code = 2;
    string name = 3;
    string status = 4;
}

message GetMenuItemRequest {
    string tenant_id = 1;
    string menu_item_id = 2;
}

message GetMenuItemResponse {
    string menu_item_id = 1;
    string menu_item_code = 2;
    string name = 3;
    string category = 4;
    int64 selling_price_cents = 5;
    string season = 6;
    string status = 7;
}

message ListMenuItemsRequest {
    string tenant_id = 1;
    string category = 2;
    string status = 3;
    string season = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListMenuItemsResponse {
    repeated GetMenuItemResponse items = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateMenuItemStatusRequest {
    string tenant_id = 1;
    string menu_item_id = 2;
    string status = 3;
    string season = 4;
}

message UpdateMenuItemStatusResponse {
    string menu_item_id = 1;
    string status = 2;
}

message CreateRecipeRequest {
    string tenant_id = 1;
    string recipe_code = 2;
    string recipe_name = 3;
    string ingredients = 4;
    string preparation_steps = 5;
    int32 yield_servings = 6;
    int32 labor_time_minutes = 7;
    string quality_standards = 8;
}

message CreateRecipeResponse {
    string recipe_id = 1;
    string recipe_code = 2;
    string recipe_name = 3;
    string status = 4;
}

message GetRecipeRequest {
    string tenant_id = 1;
    string recipe_id = 2;
    int32 version = 3;
}

message GetRecipeResponse {
    string recipe_id = 1;
    string recipe_code = 2;
    string recipe_name = 3;
    string ingredients = 4;
    int32 yield_servings = 5;
    int64 food_cost_per_serving_cents = 6;
    int32 version = 7;
    string status = 8;
}

message ListRecipesRequest {
    string tenant_id = 1;
    string status = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListRecipesResponse {
    repeated GetRecipeResponse recipes = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message CreateRecipeVersionRequest {
    string tenant_id = 1;
    string recipe_id = 2;
    string ingredients = 3;
    string preparation_steps = 4;
    int32 yield_servings = 5;
    string quality_standards = 6;
}

message CreateRecipeVersionResponse {
    string recipe_id = 1;
    int32 version = 2;
    int64 food_cost_per_serving_cents = 3;
}

message CreateShiftScheduleRequest {
    string tenant_id = 1;
    string location_id = 2;
    string schedule_date = 3;
    string shifts = 4;
    int64 labor_budget_cents = 5;
}

message CreateShiftScheduleResponse {
    string schedule_id = 1;
    string location_id = 2;
    string schedule_date = 3;
    string status = 4;
}

message GetShiftScheduleRequest {
    string tenant_id = 1;
    string schedule_id = 2;
}

message GetShiftScheduleResponse {
    string schedule_id = 1;
    string location_id = 2;
    string schedule_date = 3;
    string shifts = 4;
    double total_planned_hours = 5;
    double total_actual_hours = 6;
    string status = 7;
}

message PublishScheduleRequest {
    string tenant_id = 1;
    string schedule_id = 2;
}

message PublishScheduleResponse {
    string schedule_id = 1;
    string status = 2;
    string published_at = 3;
}

message FinalizeScheduleRequest {
    string tenant_id = 1;
    string schedule_id = 2;
    double total_actual_hours = 3;
    int64 actual_labor_cents = 4;
}

message FinalizeScheduleResponse {
    string schedule_id = 1;
    string status = 2;
}

message CreateDailySummaryRequest {
    string tenant_id = 1;
    string location_id = 2;
    string summary_date = 3;
    int64 sales_cents = 4;
    int32 guest_count = 5;
    int64 labor_cost_cents = 6;
    int64 food_cost_cents = 7;
    int64 waste_cents = 8;
    double customer_satisfaction_score = 9;
    int32 speed_of_service_seconds = 10;
    string top_selling_items = 11;
}

message CreateDailySummaryResponse {
    string summary_id = 1;
    string location_id = 2;
    string summary_date = 3;
    string summary_status = 4;
}

message GetDailySummaryRequest {
    string tenant_id = 1;
    string summary_id = 2;
}

message GetDailySummaryResponse {
    string summary_id = 1;
    string location_id = 2;
    string summary_date = 3;
    int64 sales_cents = 4;
    int32 guest_count = 5;
    double labor_cost_pct = 6;
    double food_cost_pct = 7;
    string summary_status = 8;
}

message FinalizeDailySummaryRequest {
    string tenant_id = 1;
    string summary_id = 2;
}

message FinalizeDailySummaryResponse {
    string summary_id = 1;
    string summary_status = 2;
}

message GetKPIDashboardRequest {
    string tenant_id = 1;
    repeated string location_ids = 2;
    string from_date = 3;
    string to_date = 4;
}

message GetKPIDashboardResponse {
    int64 total_sales_cents = 1;
    int32 total_guests = 2;
    double avg_labor_cost_pct = 3;
    double avg_food_cost_pct = 4;
    double avg_waste_pct = 5;
    double avg_satisfaction = 6;
    int32 avg_speed_seconds = 7;
    repeated LocationKPISummary location_summaries = 8;
}

message LocationKPISummary {
    string location_id = 1;
    string location_name = 2;
    int64 sales_cents = 3;
    double labor_cost_pct = 4;
    double food_cost_pct = 5;
    double satisfaction_score = 6;
}
```

---

## 6. Events

### Published Events
| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `restaurant.location.opened` | `restaurant.events` | `{ location_id, location_code, location_name, concept_type, region, opened_date }` | New restaurant location opened |
| `restaurant.menu.updated` | `restaurant.events` | `{ menu_item_id, menu_item_code, name, category, status, season, tenant_id }` | Menu item created or updated |
| `restaurant.shift.published` | `restaurant.events` | `{ schedule_id, location_id, schedule_date, total_planned_hours, employee_count }` | Shift schedule published to staff |
| `restaurant.daily.closed` | `restaurant.events` | `{ summary_id, location_id, summary_date, sales_cents, guest_count, labor_cost_pct, food_cost_pct }` | Daily operations summary finalized |

### Consumed Events
| Event | Source | Description |
|-------|--------|-------------|
| `inventory.threshold` | Inventory Service (34) | Ingredient stock level alerts for recipe availability checks |
| `order.completed` | External POS | Completed order data for daily summary aggregation |
| `employee.time_punched` | Time & Labor (52) | Employee clock-in/out data for shift schedule actuals |

---

## 7. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `inv-service` (34) | Real-time ingredient inventory levels and stock thresholds for recipe availability |
| `timelabor-service` (52) | Employee time punches, labor hours, and cost data for shift actuals |
| `hr-service` (47) | Employee profiles, roles, and skills for shift assignment validation |
| `auth-service` (05) | User identity and role-based access for manager and staff actions |

### Published To
| Service | Data |
|---------|------|
| `inv-service` (34) | Ingredient consumption from recipe execution, requisitions for restocking |
| `gl-service` (06) | Daily sales revenue journal entries, labor cost postings, food cost allocations |
| `loyalty-service` (132) | Guest visit events and spend data for loyalty points earning and tier evaluation |
| `hr-service` (47) | Schedule compliance data, overtime alerts, and attendance exceptions |

---

## 8. Migrations

1. V001: `rest_locations`
2. V002: `rest_menu_items`
3. V003: `rest_recipes`
4. V004: `rest_shift_schedules`
5. V005: `rest_daily_summaries`
6. V006: Triggers for `updated_at`
