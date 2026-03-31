# **FlightLog Pro \- Stable Edition**

## **Project Overview**

FlightLog Pro is a lightweight, high-performance web application designed to visualize flight route data from ZIP-compatible export files (for example `.zip` or `.effarchive`) that contain a `routes.backup` file. It was specifically developed to handle complex flight data structures and present them in a clean, user-friendly interface optimized for mobile and desktop viewing.  
The application is built as a **Single File Application (SFA)**, meaning all logic, styles, and templates are contained within a single HTML file. This ensures maximum portability and eliminates common "Script Errors" caused by CORS or external module loading in offline environments.

## **Core Functionality**

* **Archive Import:** Opens ZIP-compatible exports, extracts `routes.backup` and `flight.backup`, and parses both files.  
* **Flight Summary:** Builds the Summary tab from `flight.backup` (aircraft data, airports/runways, distances, ETOPS and dispatcher details).  
* **Operational Notes:** Displays `flight.backup` OptiClimb content, including climb guidance, passenger announcement, and ground services notes.  
* **Route Selection:** Automatically identifies the primary flight route (searching for id: 0\) while providing fallbacks for other data formats.  
* **Data Chronology:** Implements a sorting algorithm that organizes waypoints by their sequenceId to ensure the flight path is displayed in the correct order, regardless of the JSON's internal sorting.  
* **Aviation Metrics Visualization:** Extracts and formats specific aviation data displayed in WaypointCard rows:  
  * **OFF BLOCK Start Card:** The OFP tab begins with a visually highlighted OFF BLOCK card before the waypoint list. It uses `flight.backup` data to show the origin airport ICAO as the title, the scheduled off-block time from `scheduledTimes.offBlock`, and the block fuel from `plannedFuels` where `uid = "blockFuel"`.
  * **Base Columns (Always Display):** Waypoint title, planned time (HH:MM), fuel used, and fuel remaining.  
  * **Waypoint Detail Popup:** Clicking a waypoint title opens a modal with full waypoint details and all non-empty STATIC_COLUMNS values.
  * **Shared Actual Inputs:** The modal contains the ACTUAL controls (time, fuel used, fuel remaining with ▲/▼ steppers). Values are synchronized with the waypoint card in real time.
  * **Dynamic Column Selection:** Users can toggle 19 additional fields via the Settings panel (gear icon):
    * Flight Parameters: Tropopause, Indicated Air Speed, Temperature, Wind Shear, Wind Component
    * Navigation: Ground Distance, Latitude, Longitude, Track data (segment & outbound in true/magnetic)
    * Averages: Wind Speed, Minimum Safe Altitude, Ground Speed, Mach Speed, True Air Speed, ISA Deviation, Altitude
  * **Compact OFP Card Display:** The card shows planned values with inline diff in parentheses, e.g. `10:40Z (+3 min)`, `2930 KG (-100 KG)`, `44212 KG (+100 KG)`.
  * **Diff Color Coding:** Parenthesized diff is color-coded `green` for beneficial, `red` for adverse, and `blue` when diff is zero.
  * **Unit Normalization:** Automatically formats units (e.g., degrees to °, fuel to kg/lbs).

## **Technical Architecture**

### **1\. Technology Stack**

* **React (v18.2.0):** Used for state management and UI rendering. The implementation uses React.createElement (h) instead of JSX to remain compatible with standard browser environments without needing a compiler.  
* **Tailwind CSS:** Utilized for a modern, responsive "Apple-style" UI (rounded corners, subtle shadows, high whitespace).  
* **Vanilla JavaScript:** All business logic is written in modern ES6+.

### **2\. Logic Modules (Refactored)**

The code is divided into four distinct logical layers:

* **Utility Functions:** Small, pure functions for string and number formatting (formatTime, formatFuel, formatNumber, formatUnit), value extraction (getNestedValue, extractTrackValue), and deep object navigation.  
* **Data Processing:** A robust parser (processFlightData) that handles error boundaries, JSON parsing, route extraction, and waypoint sorting.  
* **UI Components:** Modular React components (WelcomeScreen, RouteHeader, OffBlockSection, WaypointCard, WaypointDetailModal) that ensure the interface is maintainable and extensible.  
* **Static Column System:** A hardcoded array (STATIC_COLUMNS) defining 19 available aviation metrics fields with human-readable labels. Users toggle column visibility through the Settings panel, which updates the component grid dynamically.  
* **App Controller:** The main App component managing the lifecycle (Upload → Process → Display → Reset) with state for route data, selected columns, and UI visibility.

### **3\. Static Columns System**

The application uses a **static, curated column system** instead of dynamic discovery:

* **STATIC_COLUMNS Definition:** Array of 19 column objects, each containing:
  * `path`: Dot-notation path to the data (e.g., `"averages.trueAirSpeed"`, `"tracks.segment.true"`)
  * `label`: Human-readable display name (e.g., `"True Air Speed (Avg)"`)
  * `defaultVisible`: Boolean (all set to `false` — users must toggle fields on)

* **Track Field Handling:** Tracks are stored as arrays with `{value, unit, type}` objects. The `extractTrackValue()` function handles special track paths:
  * `"tracks.segment.true"` → finds the object with `type: "true"` in the segment array
  * `"tracks.segment.magnetic"` → finds the object with `type: "magnetic"`
  * Same pattern for `"tracks.outbound.true"` and `"tracks.outbound.magnetic"`

* **Value Formatting Pipeline:**
  1. `getNestedValue(waypoint, path)` → detects if path is a track field and delegates to `extractTrackValue()`
  2. `formatWaypointValue()` → handles objects with `{value, unit}`, arrays, and scalar values
  3. `formatNumber()` & `formatUnit()` → locale-aware number formatting and unit string normalization

## **Available Data Fields**

The 19 configurable columns are organized into groups:

**Flight Parameters (5 fields)**
* Tropopause (value + unit)
* Indicated Air Speed (value + unit)
* Temperature (value + unit)
* Wind Shear (value + unit)
* Wind Component (value + unit)

**Navigation (8 fields)**
* Ground Distance (value + unit)
* Latitude
* Longitude
* Track Segment (True) — displays track.segment where type="true"
* Track Segment (Magnetic) — displays track.segment where type="magnetic"
* Track Outbound (True) — displays track.outbound where type="true"
* Track Outbound (Magnetic) — displays track.outbound where type="magnetic"

**Averages (6 fields)**
* Wind Speed
* Minimum Safe Altitude (value + unit)
* Ground Speed
* Mach Speed
* True Air Speed
* ISA Deviation
* Altitude

## **Data Structure Requirements**

The app expects a JSON array or object containing:

* `id`: Integer (Route identifier, 0 preferred).  
* `airport`: Object containing `name` and `icaoCode`.  
* `waypoints`: Array of objects, each containing:  
  * `sequenceId`: Integer for sorting.  
  * `title`: String.  
  * `passingByDate`: ISO Timestamp.  
  * `fuel`: Object with `remaining`, `cumulated`, `burned`, and `unit`.  
  * `groundDistance`: Object with `value` and `unit`.
  * Optional: Any of the 19 available fields listed above (each from waypoint root or nested in objects like `averages.*` or `tracks.*`)

## **Development Roadmap (Future Enhancements)**

* **Multi-Route Support:** Add a selector to switch between main and alternate routes.  
* **Map Integration:** Use waypoint coordinates (latitude, longitude) to render a 2D/3D flight path.  
* **Offline Persistence:** Implement local storage to keep the last uploaded flight log active after a page refresh.  
* **Export Functionality:** Generate PDF or CSV reports with selected columns for archival or sharing.  
* **Advanced Analysis:** Flight profile comparisons, performance metrics, and anomaly detection.

## **Fuel Check — EFB Workflow**

A core EFB (Electronic Flight Bag) function is the **fuel check**: at each waypoint the pilot compares the planned fuel values from the OFP against what is actually observed on the aircraft fuel gauges. FlightLog Pro implements this workflow end-to-end.

### How It Works

1. **Planned values** are parsed from `routes.backup` (`fuel.cumulated` and `fuel.remaining` for each waypoint) and displayed on every `WaypointCard` in the OFP tab.

2. **Actual entry** — The pilot taps a waypoint title to open the `WaypointDetailModal`, which exposes three editable fields:
   - **Time** — actual UTC passing time (HH:MM)
   - **Used** — cumulative fuel burned to this waypoint (KG or LBS)
   - **Remaining** — fuel on board at this waypoint (KG or LBS)

3. **Stepper buttons (▲/▼)** allow quick adjustment in **100-unit** increments (`FUEL_STEP = 100`). On the first tap, the field initialises to the nearest 100-unit boundary around the planned value so the pilot can make a small adjustment rather than typing from scratch.

4. **Diff calculation** — All diffs follow `Actual − Planned`:
   - *Time diff*: positive = late, negative = early
   - *Fuel Used diff*: positive = more burned than planned (adverse)
   - *Fuel Remaining diff*: positive = more fuel on board than planned (beneficial)

5. **Colour coding** makes deviations immediately scannable:
   | Colour | Meaning |
   |--------|---------|
   | 🔴 Red | Adverse deviation (late, burned more) |
   | 🟢 Green | Beneficial deviation (early, saved fuel) |
   | 🔵 Blue | Exactly on plan (zero diff) |
   > Exception: for *Fuel Remaining* the sign is inverted before colouring — more fuel remaining is good, so a positive diff is shown in green.

6. **Card annotations** — After entering actuals, the `WaypointCard` shows the planned value with the diff inline, e.g.:
   - `10:40Z (+3 min)` in red — 3 minutes late
   - `2930 KG (-100 KG)` in green — 100 KG less burned than planned
   - `44212 KG (+100 KG)` in green — 100 KG more remaining than planned

7. **State sync** — `actualData` is owned by `WaypointCard` and passed into `WaypointDetailModal`, so values edited in the modal are immediately reflected on the card without any extra save step.

---

## **How to Use**

1. **Upload Flight Data:** Click "Choose File" on the welcome screen and select a `.zip`/`.effarchive` export containing `routes.backup` and `flight.backup`.
2. **View Flight Plan:** The application displays a highlighted OFF BLOCK start card first, followed by waypoints in chronological order with base columns (Time, Used Fuel, Remaining Fuel).
3. **Review OFF BLOCK Snapshot:** The special OFP start card shows the departure airport ICAO, scheduled off-block time, and planned block fuel from `flight.backup`.
4. **Configure Columns:** Click the **Settings (gear) icon** in the top-right to open the Column Settings panel.
5. **Toggle Fields:** Check/uncheck any of the 19 available fields to add or remove columns from the waypoint grid (all default to hidden).
6. **Open Waypoint Popup:** Click a waypoint title to open the detail popup.
7. **Enter Actual Data:** In the popup, edit actual flight times and fuel values with the blue inputs and steppers.
8. **Review Compact Diff in OFP Card:** The card shows planned values with inline diff in parentheses:
  - **Green** = Beneficial deviation
  - **Red** = Adverse deviation
  - **Blue** = No deviation (zero diff)
9. **Close Popup / Continue:** Close the popup to keep reviewing the OFP cards with synchronized values.
10. **Close Flight:** Click the "Close" button to reset and upload a new file.

## **How to Run**

Simply open `index.html` in any modern web browser. No local server or installation is required. For best results, use:

* **Desktop:** Chrome, Safari, Firefox, Edge (latest versions)
* **Mobile:** iOS Safari, Chrome on Android

*Note: The application is fully self-contained in a single HTML file and works offline.*