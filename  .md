# Footmark

A personal carbon footprint tracker that helps an individual **understand, track, and reduce** their daily emissions through simple logged actions and insights drawn from their own data.

Footmark is a single, self contained `index.html`. There is no build step, no backend, and no account. Open the file in a browser and it runs. All data stays on the device in `localStorage`.

---

## 1. Chosen vertical

**Personal sustainability for individuals.**

The brief was:

> Design a solution that helps individuals understand, track, and reduce their carbon footprint through simple actions and personalized insights.

Footmark targets a single person who wants to see the climate impact of everyday choices (a commute, a meal, an online order, an evening of electricity use) and to bring that impact down over time. The product is built specifically around an Indian household, so the energy model uses the India grid emission factor rather than a generic global one.

The three verbs in the brief map directly onto the product:

* **Understand.** Every number is shown in plain context. A day's total is expressed in kg CO2e, but also as an equivalent car distance and as the number of trees needed to offset a year at that rate. A footnote anchors the figure against the Paris pathway and the global average.
* **Track.** Entries persist across days. A seven day trend chart, a running streak of days under target, and a per category pie chart turn one off logging into a record that builds over time.
* **Reduce.** The user sets a daily target. A progress ring, a streak counter, and category specific tips push gently toward a lower number, and the insights call out the single largest source each day.

---

## 2. What it does

Footmark logs activity across four categories and rolls everything into a daily footprint.

* **Transport.** Pick a mode (car by fuel type, motorcycle, bus, train, flight, bicycle) and a distance. Emissions scale by a per kilometre factor.
* **Food.** Type what you ate in plain words, for example "chicken rice and salad". The text is matched against a food carbon database, each recognised component is added, and the result is shown as pills with calories alongside the carbon. Calories and carbon are presented side by side as separate estimates. One is not derived from the other, because food carbon depends on the food type rather than its energy content.
* **Energy.** Enter grid electricity used in kWh. A "rooftop solar" checkbox reveals a second field for the share supplied by solar. Only the grid portion is counted, since solar generation carries almost no operating emissions, and the app reports how much carbon the panels avoided.
* **Shopping.** Tick everything bought from a checklist, then choose **online** or **in store**. Online adds a last mile delivery figure plus a small data and server footprint. In store adds the travel emissions for how you got there (walk, cycle, bus, car, motorcycle) and the round trip distance.

Around the log sit the standing signals: a daily target you can edit, a progress ring, a streak with flame icons, a category pie chart with a legend, a seven day trend with the target marked, and a set of personalized insight cards.

---

## 3. Approach and logic

### Why a single file

The deliverable is one HTML file by design. It is the most portable form a web project can take: it opens anywhere, has zero dependencies to install, and cannot suffer dependency drift. The trade off is that all layers (markup, styling, logic, tests) live together, so the code is organised into clearly separated modules inside the one script to keep it readable and maintainable.

### Emission model

Emissions are expressed in kg CO2e per unit of activity. The factors are rounded reference figures drawn from public datasets:

* the **CEA India** grid emission factor for electricity (about 0.71 kg CO2e per kWh),
* **UK DEFRA** conversion factors for transport and shopping,
* **Our World in Data** for the per serving food figures.

These are approximations chosen to be directionally honest rather than laboratory precise, and the interface says so plainly in its footnote. The goal is to help someone compare choices and see trends, not to produce audit grade accounting.

### Food matching

A free text meal is lowercased and scanned against a keyword database. Each database entry carries a set of keywords, a carbon figure, and a calorie figure for one serving. Every entry whose keywords appear contributes one serving, the components are summed, and the matched names become pills. If nothing matches, the meal falls back to a single "Mixed meal" estimate, so the input is never rejected.

### Energy and solar

Net grid use is the consumption minus the solar share, floored at zero so the footprint can never go negative, and capped so solar can never exceed what was actually consumed. The grid portion is multiplied by the India grid factor. The solar portion is treated as near zero operationally and surfaced as the amount of carbon avoided.

### Shopping channel

A shopping total is the embodied carbon of the chosen items, plus one of two overheads. Online adds a fixed delivery figure and a small digital figure per order. In store adds the chosen travel mode factor multiplied by the round trip distance, which is zero for walking or cycling.

### Targets, streak, trend, insights

A daily target frames the footprint as progress rather than a raw number. The streak counts consecutive prior days that stayed at or under target (today is excluded because the day is still open, and a gap or an over target day ends the run). The trend reads the last seven stored days. The insights are generated from the user's own numbers: standing versus target, the largest category with a matching tip, a tangible equivalence, a calorie summary when meals were logged, and either momentum on a streak or a yearly projection with trees to offset.

---

## 4. How the solution works (architecture)

The script is a set of small single responsibility modules. Data flows one way: the UI reads from the store, calls the calculation layer to derive values, and paints. Mutations go back through the store, which persists and triggers a single batched render on the next animation frame.

| Module | Responsibility |
| --- | --- |
| `CONFIG` and reference data | Immutable emission factors, food and shopping catalogues, category colours. |
| `Calc` | Pure calculation. No DOM, no storage. Every business rule lives here so the tests can verify it in isolation. |
| `Store` | `localStorage` persistence and in memory state, guarded so a private context or a parse error never breaks the app. |
| `Insights` | Pure. Turns the user's numbers into short, specific observations. |
| `Sample` | Generates a realistic demo week on request. |
| `UI` | The only module that touches the DOM. Caches element references once, builds structure, and renders. |
| `Tests` | A zero dependency assertion harness over the pure modules and the rendered DOM. |

### Persistence

State is stored under a single versioned key (`footmark.v3`) as JSON: a map of days keyed by date, plus the daily target. Reads and writes are wrapped in try and catch, so if storage is unavailable the app simply starts fresh instead of failing.

### Rendering

Rendering favours efficiency. Element references are cached at startup. Lists are built with a `DocumentFragment` and committed once. Interactions use event delegation, so listeners are never reattached on each render. Renders are coalesced into a single animation frame, so several rapid changes paint once. The pie chart memoizes on a signature of its slices and skips rebuilding when nothing relevant changed. The food input is debounced so matching does not run on every keystroke.

### Layout

On wider screens the logo bar and the pie chart band are pinned in a fixed head, and the rest of the page scrolls beneath them, so the day summary stays in view. Below 760 pixels the log sections and review cards become a native accordion (collapsed by default, the first one open), so the page stays compact on a phone.

---

## 5. Assumptions

* **Factors are rounded estimates.** They are good enough to compare choices and reveal trends, not to certify a precise footprint.
* **One serving per matched food.** Typing "rice" adds one serving of rice. Quantities are not parsed from the text.
* **Calories are indicative.** They are shown to give a sense of the meal, not to drive the carbon figure.
* **India grid factor.** Electricity uses the CEA India average. A household on a greener tariff or in another country would differ.
* **Solar is self reported.** The user enters how much of their usage solar covered, and it is capped at total consumption.
* **Single user, single device.** There is no backend or login. Data lives in the browser's `localStorage` on that device, which keeps it private but means it does not sync across devices and can be cleared by the browser.
* **A day is open until it ends.** The streak only counts completed prior days, so today never breaks or extends it until tomorrow.

---

## 6. How your work is evaluated (focus areas)

A short note on how the project addresses each review area.

**Code quality.** One way data flow across small, documented modules. All business rules sit in a pure `Calc` layer with JSDoc. Named constants instead of magic numbers. No dead code or duplicated declarations. The rendering and state concerns are kept separate from the calculation concerns.

**Security.** A strict Content Security Policy (`default-src 'none'` with narrow allowances for the Google Fonts stylesheet and font files, data images, and the inline script and style), plus `frame-ancestors`, `base-uri`, and `form-action` locked down. No external scripts beyond the font stylesheet, and no `eval`. Every place untrusted input could appear (the free text meal, item names) is written through `textContent`, never `innerHTML`. Storage access is defensive.

**Efficiency.** Batched renders on one animation frame, a debounced text input, a memoized chart redraw, event delegation, cached element references, and fragment based list building.

**Testing.** A built in suite of 51 assertions. Forty six are pure logic tests covering the calculation and insight rules, including edge cases (zero quantities, unknown foods, solar exceeding consumption, the streak ignoring today). Five are read only DOM smoke tests that confirm the interface rendered. Open the page with `#test` at the end of the URL to see a pass or fail panel. Results also log to the browser console on every load.

**Accessibility.** A single `h1` with `h2` section headings for a clean outline. Every input is associated with a label. The progress ring and charts expose `role` and live, data rich `aria-label` summaries. The mobile accordion uses native `details` and `summary`, so keyboard and screen reader support are built in. Focus is always visible, reduced motion is respected, and colours meet contrast guidance.

---

## 7. Running it

There is nothing to install.

* **Locally.** Download `index.html` and open it in any modern browser.
* **Tests.** Open the page with `#test` appended to the URL (for example `index.html#test`) to reveal the test panel. The same results print to the browser console.
* **Demo data.** Use the "Load sample week" button to populate a week of history so the trend and streak are visible immediately.

### Optional: a live demo on GitHub Pages

Because the project is a single static file, you can publish a live version for free. In the repository settings, open **Pages**, set the source to the `main` branch and the root folder, and GitHub will serve `index.html` at a public URL you can add to your submission.

---

## 8. Project structure

```
.
├── index.html      The entire application: markup, styles, logic, and tests
└── README.md       This file
```

---

## 9. Tech stack

Vanilla HTML, CSS, and JavaScript. No frameworks, no build tooling, no runtime dependencies. The only external resource is the Google Fonts stylesheet for the two typefaces.
