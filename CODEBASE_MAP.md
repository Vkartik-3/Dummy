# FleetFlow codebase map

Reviewed on October 6, 2026. This describes the current implementation; source code takes precedence over the README. No application behavior was changed during this review.

## Architecture and startup

- Two independent npm applications; the root only has an empty package lock.
- Frontend: React 19, React Router 7, Vite 7, Axios, Recharts, Lucide, jsPDF, jspdf-autotable, html2canvas. Framer Motion and date-fns are declared but not used by the current application source.
- Backend: CommonJS, Express 4, Mongoose 8, express-validator, JWT, bcryptjs, Nodemailer.
- Request flow: React page → `frontend/src/services/api.js` → backend route → authentication/authorization → request validator → controller → Mongoose model.
- `backend/server.js` loads dotenv, calls `connectDB()`, then starts listening without awaiting the connection. `backend/src/config/db.js` reads `MONGO_URL` and exits on connection failure.
- `backend/src/app.js` configures JSON parsing, cookies, credentialed CORS for localhost ports 5173/5174, a root health response, eight API routers, and the global error handler.
- Frontend API base is `VITE_API_URL` or `http://localhost:3000/api`. Vite configuration enables the React plugin without a proxy.
- Backend env keys: PORT, NODE_ENV, MONGO_URL, JWT_SECRET, EMAIL_USER, EMAIL_CLIENT_ID, EMAIL_CLIENT_SECRET, EMAIL_REFRESH_TOKEN. Local values were not copied into this document.

## Backend file inventory

Each module below has a corresponding `backend/src/routes/<module>.routes.js` and `backend/src/controllers/<module>.controller.js`.

| Module | Current behavior |
| --- | --- |
| auth | Register, login, current profile, update name/phone, logout, forgot-password notification. Seven-day JWT in an HTTP-only cookie and response body. |
| vehicles | CRUD, stats, filtered/searchable/sortable paginated list, manual available/retired status changes. Deletes blocked for on-trip/in-shop vehicles. |
| drivers | CRUD, paginated search/status filtering, populated assigned vehicle, performance endpoint. Deletes blocked for on-trip drivers. |
| trips | Create/edit draft, list/detail, dispatch/complete/cancel. Changes driver and vehicle state and trip counters; completion records odometer and optional driver rating. |
| maintenance | Create/list/detail/update/delete and per-vehicle history. Creation sets vehicle in_shop; completion or scheduled deletion releases it. Completed records cannot be edited. |
| expenses | Create/update/delete/list, per-vehicle list, per-vehicle aggregated category totals. Optional trip reference; fuel requires liters on creation. |
| dashboard | Fleet/trip/driver/maintenance counts, total expense amount, six-month trip/expense record counts, recent trips and maintenance. Some failures are swallowed by legacy optional-module catches. |
| analytics | Date-filtered fuel efficiency, vehicle ROI, cost/km, and CSV exports of vehicles/maintenance/trips/expenses. Uses sequential per-vehicle aggregation queries. |

`backend/src/models/` contains six schemas:

- `user.model.js`: unique normalized email, hidden hashed password, name, phone, role, active flag; pre-save bcrypt hashing and password comparison method.
- `vehicle.model.js`: unique uppercased plate, name/model/type, capacity, odometer, status, fuel type, acquisition cost/date, year/color/region/notes, creator. Status/type and region indexes.
- `driver.model.js`: unique license number, expiry, contact info, safety score, average rating/count, status, assigned vehicle, total/completed trip counters, creator. License-validity and completion-rate virtuals; save hook requires an assigned vehicle for on_trip.
- `trip.model.js`: vehicle/driver refs, cargo, route, planned distance, status, starting/ending odometer/date, notes, rating, creator. Vehicle/status, driver/status and start-date indexes.
- `maintenance.model.js`: vehicle ref, service category, description, cost, service/completion date, status, mechanic/notes, creator. Vehicle/status and date indexes.
- `expense.model.js`: vehicle and optional trip refs, category, amount, liters, expense date/notes, creator. Vehicle/type, trip, and date indexes.

Other backend files:

- `middleware/auth.middleware.js`: cookie token preferred over bearer header, JWT verification, user lookup, active-account check, and `authorize(...roles)`.
- `middleware/error.middleware.js`: formats Mongoose/JWT/default errors; most controllers catch errors directly, bypassing it.
- `validators/`: auth, vehicle, driver, trip, maintenance, and expense rule files plus `validate.js`, which returns field/message errors with HTTP 400. Validation does not strip unspecified fields.
- `utils/constants.js`: role, vehicle type/status, driver status, trip status, maintenance status/service type, expense type and fuel type enums.
- `services/email.service.js`: Gmail OAuth2 transport, verification on module load, registration/login/forgot-password HTML notifications. Send errors are logged and swallowed.
- `services/export.service.js`: CSV generation with nested property lookup and comma/quote/newline escaping; downloadable response headers.
- `backend/seed.js`: manager, three vehicles/drivers, six expenses, four trips, three maintenance logs. Existing users/assets are reused; other collections are skipped if nonempty. Direct inserts bypass controller state synchronization. Do not run merely to inspect the application.

## Frontend file inventory

- `src/main.jsx`: StrictMode root and global CSS.
- `src/App.jsx`: public auth pages and protected dashboard/profile/module routes; role restrictions wrap the shared layout.
- `src/context/AuthContext.jsx`: authentication loading/user state, login/register/logout, `/auth/me` restoration and role helpers. Restoration requires a localStorage token even when a cookie exists.
- `src/services/api.js`: credentialed Axios instance plus bearer token request interceptor. No global response interceptor.
- `src/components/layout/ProtectedRoute.jsx`: loading/login/access-denied handling, Sidebar/Navbar/Outlet layout and pathname-derived title.
- `src/components/layout/Sidebar.jsx`: all module links, profile link, current user summary, logout. Links are not filtered by role.
- `src/components/layout/Navbar.jsx`: page title.
- `src/components/common/{Button,Input,Table}.jsx`: button variants/loading, labeled/icon inputs, generic rows/loading/empty table.

| Page file under `src/pages/` | Data/UI behavior |
| --- | --- |
| Login.jsx | Auth-context login and dashboard redirect. |
| Register.jsx | Auth-context registration; role selector defaults to manager. Phone is required in the UI. |
| ForgotPassword.jsx | Calls notification endpoint while describing a reset-link flow. |
| Dashboard.jsx | `/dashboard`, four KPI cards, monthly counts bar chart, fleet pie chart, unfiltered quick-action links. |
| Vehicles.jsx | Server search and filters, 10-row pagination, `/vehicles/stats`, manager CRUD forms, delete dialog and toast. No manual retirement UI. |
| Drivers.jsx | Debounced server search/status filter, always first 10 records, add/edit/delete dialogs, license/performance/rating display. |
| Trips.jsx | Server status filter/pagination, current-page client search, four count queries, available/on-duty dropdowns, draft edit, dispatch, completion odometer/rating, cancel. |
| Maintenance.jsx | Server category/status filters/pagination, current-page client search, counts, vehicle dropdown, manager create/edit/start/complete/delete. |
| Expenses.jsx | Server category/vehicle filters/pagination, current-page client search, counts, vehicle dropdown, create/edit/delete forms. |
| Analytics.jsx | Three analytics APIs plus first vehicle page; date and type selectors; seven charts; client PDF capture and data tables. |
| Profile.jsx | Current auth user display, name/phone update, `/auth/me` check then full reload. |

Styles: `src/index.css` defines the dark theme, Inter font, glass cards, resets and animations. Layout/common components have matching CSS. Page styles exist for Auth, Dashboard, Vehicles, Drivers, Trips, Maintenance, Expenses, Analytics and Profile. `Vehicles.css` provides shared CRUD/modal/filter/toast styles; selectors are global and overlap with Drivers/Profile/Dashboard styles. Layout fixes a 260px sidebar and 70px navbar; only some chart grids have responsive breakpoints. `App.css` and React/Vite SVG assets remain from the starter template; App.css is not imported. `index.html` still uses the Vite favicon and title "frontend".

## Permissions as implemented

| Area | Backend reads | Backend writes | Frontend page access |
| --- | --- | --- | --- |
| Vehicles | All four roles | Manager | All four roles |
| Drivers | All four roles | Manager/dispatcher; delete manager | Manager/dispatcher |
| Trips | All four roles | Manager/dispatcher | Manager/dispatcher |
| Maintenance | All four roles | Manager | Manager/safety officer |
| Expenses | Manager/financial analyst/dispatcher | Same; delete manager | Manager/financial analyst |
| Analytics | Manager/financial analyst | CSV downloads only | Manager/financial analyst |
| Dashboard/profile | All four roles | Own profile | All four roles |

## Cross-module workflows

- Draft creation checks vehicle availability/capacity and driver duty/license. Drafts do not reserve resources.
- Dispatch rechecks vehicle/driver status, saves starting odometer/date, marks both on_trip, assigns vehicle and increments totalTrips.
- Completion requires end odometer, releases both resources, updates vehicle odometer, increments completedTrips and optionally updates driver average rating.
- Cancellation releases dispatched resources and decrements totalTrips; draft cancellation only changes the trip.
- Maintenance creation immediately blocks a vehicle, including future scheduled records. Completion releases it and sets completionDate.
- Fuel efficiency = completed odometer distance / logged fuel liters. ROI = negative operational cost / acquisition cost; revenue is not implemented. Cost/km includes maintenance log costs and fuel expense amounts only.

## Confirmed gaps and risks for future work

1. `expense.controller.js` declares `getAllExpenses` twice; the later declaration wins and returns nested `pagination`, while Expenses.jsx reads top-level `total` and `totalPages`. Result counts, stat counts and pagination therefore fall back incorrectly. The expense router also registers GET `/` twice.
2. Forgot-password has no reset token storage, reset URL, password update endpoint, or reset page.
3. Public registration accepts any role, including manager, without an approval/invitation check. Whether this is intended needs product context before changing it.
4. Dispatcher sees driver deletion although the API allows only managers. Financial analysts can update expenses through the API, but frontend edit controls are manager-only. Dispatcher expense API access has no matching frontend page access.
5. Dispatch does not recheck license expiry or cargo capacity after draft creation. Multi-document status saves are not transactional or guarded against concurrent dispatches.
6. Multiple active maintenance records are allowed; completing/deleting one releases the vehicle even if another remains. Retired vehicles can be sent to maintenance and subsequently restored to available.
7. Driver updates pass the body directly to findByIdAndUpdate, including schema fields not validated by the route. Query updates bypass the driver's pre-save assignment check.
8. Vehicle/driver deletions can leave historical/draft references unresolved. Dispatch assumes referenced documents still exist.
9. Trip/maintenance/expense searches operate only on the fetched page. Drivers has no pagination UI, and its KPI cards summarize the fetched subset.
10. Analytics status distribution uses only the default first 10 vehicles, ignores type/date filters, and omits bike from the type selector. Type-cost summary uses unfiltered costData.
11. Analytics date upper bounds parse to midnight, excluding later events on the selected final day. ROI may double-count maintenance if also entered as an expense; cost/km omits non-fuel expense categories.
12. Profile fields isActive/createdAt are absent from login/register responses until `/auth/me` runs. Optional blank phone/email values can fail request validation when explicitly submitted.
13. README uses MONGODB_URI and SMTP password settings; code uses MONGO_URL and Gmail OAuth2. README also describes unimplemented driver status PATCH, expense detail GET, and user management.
14. No automated test suite is configured. Backend test script is a failing placeholder. Postman collection is a manual request collection, not complete regression coverage.

## Verification baseline

- Frontend production build passed; main JS bundle is approximately 1.42 MB before gzip and triggers the chunk-size warning.
- All 38 backend JavaScript files passed `node --check`.
- ESLint over `frontend/src` reported 10 errors and 2 warnings: Fast Refresh export rules, unused bindings (including Icon bindings used in JSX), and effect dependency warnings.
- Full `npm run lint` reported 1108 errors and 11 warnings because it also scans generated `.vite/deps` files; the config only ignores dist.
- Database integration, email delivery and browser workflows were not exercised. No server, seed, database writes, or outgoing emails were triggered.
- Package locks were checked for structure; installed dependencies, generated caches and bundled third-party code were not treated as application source.
