### Routing Logic

The app uses **hash-based client-side routing** with no external router library. `App.jsx` listens to `window.hashchange` events and maps URLs to views:

| URL Hash | View | Role Required |
|----------|------|--------------|
| `#/` | Home | Public |
| `#/login` | Login | Public |
| `#/dashboard` or `#/manager-dashboard` | Manager Dashboard | `manager` or `admin` |
| `#/driver` or `#/driver-dashboard` | Driver Dashboard | `driver` |
| `#/export-hub` | Export Hub | `manager` or `admin` |

---

## 👥 User Roles & Demo Credentials

| Role | Email | Password | Redirected To |
|------|-------|----------|--------------|
| `admin` | `admin@fleetops.com` | `admin123` | Manager Dashboard |
| `manager` | `manager@fleetops.com` | `manager123` | Manager Dashboard |
| `driver` | `driver@fleetops.com` | `driver123` | Driver Dashboard |

> 💡 Use the **Quick Demo Access** buttons on the Login page for instant one-click access.

---

## 📄 Pages & Components

### 🏠 Home Page
Assembled from 6 reusable sections: `Hero`, `WhyFleetOps`, `OperationalHighlights`, `Workflow`, `Contact`, and `Footer`. Fully public, no authentication required.

---

### 🔐 Login Page (`Login.jsx`)
- Email + password form with show/hide password toggle
- Calls `POST /api/auth/login` via the `api.js` layer
- JWT token stored in `localStorage` on success
- Auto-redirects by role: managers → dashboard, drivers → driver portal
- Three **Quick Demo Access** buttons (Admin / Manager / Driver) for instant login

---

### 📊 Manager Dashboard (`ManagerDashboard.jsx`)

The Fleet Operations Control Panel — the core of the application.

#### KPI Cards
Four summary cards computed from live vehicle data using `useMemo`:
- **Total Fleet** — total vehicles tracked
- **Active (En Route)** — count with active utilization percentage
- **Idle (Available)** — count of depot-ready vehicles
- **Under Maintenance** — count with active alert badge showing issue count

#### Vehicle Table
- **Search bar** — searches across vehicle ID, driver name, vehicle type, location, and destination simultaneously
- **Filter tabs** — All / En Route / Idle / Maintenance (active tab highlighted in blue)
- **Per-row display**: Vehicle ID (with `!` alert badge) · Driver + Type · Status badge with animated dot · Fuel % + progress bar · Speed / Location · Chevron arrow
- Click any row to open the Telematics Drawer
- Footer shows count of visible vs total assets + Autopoll status

#### Live Telemetry Simulator
Toggle button that runs a `setInterval` every **4.5 seconds** simulating real fleet events:

| Probability | Event Type |
|-------------|-----------|
| 70% | Minor update — speed adjusts ±4 mph; idle vehicle has 20% chance to auto-dispatch |
| 30% | Major event — status flip, low fuel alert trigger, or alert resolution |

All simulated changes are synced to the backend via `api.updateVehicleStatus()` and logged via `api.addLog()`. Local state updates optimistically without waiting for the server.

#### Live Event Feed (Sidebar)
- Fixed-height scrollable panel (520px) showing the last 20 logged events
- Color-coded dots: 🔴 warning · 🟢 success · 🔵 info
- Vehicle IDs in the feed are clickable — opens that vehicle's drawer
- **Clear Feed** button resets the log terminal

#### Telematics Drawer
Slide-in panel from the right (`animate-in slide-in-from-right`) with a blurred backdrop overlay. Opens when any table row is clicked. Contains:

| Section | Details |
|---------|---------|
| **Driver Info** | Assigned driver name + vehicle class type |
| **Active Warning Codes** | Red alert panel listing all current alerts (only shown if alerts exist) |
| **Manager Quick Override** | 3 buttons to instantly change vehicle status (En Route / Idle / Maintenance) — sets speed and alerts automatically |
| **Engine Diagnostics** | Fuel %, Speed, Coolant Temp (turns red if >210°F), Tire Pressure, Cargo Load |
| **Telemetry Routing Map** | Dark SVG mock-map with dashed route lines, animated pulsing truck icon, HUD bar showing Location → Destination |
| **Footer** | Export Diagnostics button |

#### Toast Notifications
Stack top-right, auto-dismiss after 4 seconds. Three styles: warning (red), success (emerald), info (blue).

---

### 🚛 Driver Dashboard (`DriverDashboard.jsx`)

The driver's personal operations portal.

- **Vehicle Selection** — dropdown listing only `Idle` vehicles available for assignment
- **Shift Management** — Start Shift sets vehicle to `En Route`; End Shift sets it back to `Idle`
- **Live Shift Timer** — counts up HH:MM:SS while shift is active
- **Live Telemetry Panel** — shows current vehicle's real-time speed, fuel level, and engine temperature
- **Incident Reporting** — form to submit incidents with type, severity, location, and description; creates a vehicle alert + log entry on the backend
- **Shift History** — table of all past shifts with start/end times and duration
- **Toast Notifications** — success/error feedback for all actions

---

### 📋 Export Hub (`ExportHub.jsx`)

Role-guarded report generation panel (managers and admins only).

- Select date range: **Current Week** / **Today** / **Last 30 Days**
- Triggers `GET /api/reports/export-csv` with role header
- Downloads response as a `.csv` file directly in the browser
- Shows a **403 Security Guardrail** block with detailed middleware info if accessed by an unauthorized role
- Footer references ISO 27001 & FMCSA compliance standards

---

## ⚙️ Backend — `server.js`

Single-file Express server with MongoDB (Mongoose). On startup with a valid `MONGODB_URI`, it automatically **seeds** the database with default users, 8 vehicles, event logs, and report data if the collections are empty.

### REST API Endpoints

#### Health
| Method | Route | Description |
|--------|-------|-------------|
| `GET` | `/` | API health check |
| `GET` | `/api/health` | Returns DB connection state + uptime |

#### Authentication
| Method | Route | Body | Description |
|--------|-------|------|-------------|
| `POST` | `/api/auth/login` | `{ email, password }` | Returns user object on success |

#### Vehicles
| Method | Route | Description |
|--------|-------|-------------|
| `GET` | `/api/vehicles` | Returns all vehicles |
| `PUT` | `/api/vehicles/:id/status` | Update status, speed, fuel, telemetry, alerts |
| `POST` | `/api/vehicles/:id/select` | Assign a driver to the vehicle |
| `POST` | `/api/vehicles/:id/deselect` | Remove the assigned driver |

#### Shifts
| Method | Route | Body | Description |
|--------|-------|------|-------------|
| `GET` | `/api/shifts` | — | Returns all shifts |
| `POST` | `/api/shifts/start` | `{ driverName, vehicleId }` | Starts a shift, sets vehicle to En Route |
| `POST` | `/api/shifts/end` | `{ shiftId }` | Ends active shift, sets vehicle to Idle |

#### Incidents
| Method | Route | Body | Description |
|--------|-------|------|-------------|
| `GET` | `/api/incidents` | — | Returns all incidents |
| `POST` | `/api/incidents` | `{ vehicleId, driverName, type, severity, location, description }` | Creates incident + adds alert to vehicle + logs event |

#### Logs
| Method | Route | Body | Description |
|--------|-------|------|-------------|
| `GET` | `/api/logs` | — | Returns last 20 logs (newest first) |
| `POST` | `/api/logs` | `{ vehicleId, event, type }` | Adds a new log entry |

#### Reports (Manager/Admin only)
| Method | Route | Query | Description |
|--------|-------|-------|-------------|
| `GET` | `/api/reports/export-csv` | `range`, `role` | Returns CSV file download; role verified via `verifyRole` middleware |

### Role Middleware
```js
verifyRole(['admin', 'manager'])
```
Reads the `x-user-role` header or `role` query param. Returns `401` if missing, `403` if unauthorized.

---

## 🗄️ Database Models

### User
