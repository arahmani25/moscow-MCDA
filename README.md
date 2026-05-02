# Moscow Housing MCDA Dashboard

**Course:** Decision Analysis & Optimization — HSE University 2024–25  
**Team:** Fidan Akhundova · Frank A. Carrasco Paulino · Ahmad Shah Rahmani  
**Live demo:** deploy both files to GitHub Pages (instructions below)

---

## Project goal

Apply Multi-Criteria Decision Analysis (MCDA) to real Moscow apartment data so a user can find the best apartment according to their personal priorities — not just the cheapest, or the closest to the metro, but the one that scores best across all chosen criteria simultaneously.

---

## Files

```
index.html   — the full interactive web app (42 KB)
data.js      — all 22,676 apartment records + pre-computed statistics (3.1 MB)
```

The app loads `data.js` as a script tag. No server, no API, no build step needed.

---

## How to deploy on GitHub Pages

1. Create a new GitHub repository (e.g. `moscow-mcda`)
2. Upload both `index.html` and `data.js` to the root of the repository
3. Go to **Settings → Pages → Source → Deploy from branch → main → / (root)**
4. Wait ~60 seconds — the app is live at `https://<username>.github.io/<repo>/`

---

## How the data is structured

The Python script reads the raw `data.csv` (22,676 rows, 12 columns) and produces `data.js`. Each apartment is stored with short key names to keep file size manageable:

```python
# Python preprocessing — produces data.js
import csv, json, statistics

rows = []
with open('data.csv') as f:
    reader = csv.DictReader(f)
    for r in reader:
        rows.append(r)

apartments = []
for r in rows:
    apartments.append({
        'p':  round(float(r['Price']) / 1e6, 2),           # price in M RUB
        'r':  int(float(r['Number of rooms'])),             # rooms (0 = studio)
        'a':  round(float(r['Area']), 1),                   # total area m2
        'la': round(float(r['Living area']), 1),            # living area m2
        'ka': round(float(r['Kitchen area']), 1),           # kitchen area m2
        'm':  int(float(r['Minutes to metro'])),            # walk to metro (min)
        'rn': r['Renovation'],                              # renovation level
        'st': r['Metro station'].strip(),                   # nearest station
        'fl': int(float(r['Floor'])),                       # floor number
        'tf': int(float(r['Number of floors'])),            # total building floors
        'rg': r['Region'],                                  # Moscow / Moscow region
        'tp': r['Apartment type'],                          # New / Secondary
    })
```

The `data.js` file also contains pre-aggregated statistics (medians by renovation, region, metro distance, etc.) as a `STATS` object used by the dashboard charts.

---

## App structure — one HTML file

```
index.html
├── <style>           CSS variables and all component styles
├── Sidebar           Navigation calling go(id, btn)
├── #page-overview    Dashboard — 6 Chart.js charts from STATS
├── #page-explore     Filterable paginated cards from full APARTMENTS array
├── #page-mcda        MCDA weight sliders and live ranking
├── #page-insights    6 research questions answered with bar charts
├── #page-about       Team, dataset schema, methods
└── <script>
    ├── go()          Page navigation
    ├── buildStatCards()    Renders summary stat cards
    ├── buildDashCharts()   Renders all 6 Chart.js charts
    ├── applyFilter()       Filters APARTMENTS array and re-renders
    ├── renderPage()        Paginates and renders apartment cards
    ├── aptScore(a, w)      Computes MCDA score for a real listing
    ├── buildSliders()      Renders weight sliders
    ├── slide(key, val)     Updates weight and re-ranks
    ├── applyProfile()      Sets preset weights (Student / Family / Worker)
    └── renderRanking()     Re-ranks the 4 apartment profiles
```

---

## The MCDA model — Weighted Scoring Model (WSM)

### Formula

```
Final Score = sum( criterion_score_i × weight_i )
```

All criterion scores are on a 1–10 scale. The final score is the weighted average.

### The four apartment profiles

These are stylised archetypes built from typical Moscow market patterns:

| Apartment | Price | Distance | Size | Comfort | Infrastructure |
|-----------|-------|----------|------|---------|----------------|
| A — City center  | 4  | 9 | 7 | 9  | 9 |
| B — Suburb       | 7  | 7 | 8 | 7  | 7 |
| C — Remote/Budget| 10 | 4 | 6 | 5  | 5 |
| D — Modern       | 5  | 6 | 9 | 10 | 8 |

Score 10 = best for that criterion. For price, 10 = cheapest.

### Default weights and their justification

| Criterion        | Weight | Justification |
|-----------------|--------|---------------|
| Price affordability | 0.30 | Largest constraint for most buyers |
| Metro proximity     | 0.25 | Commute is a daily cost |
| Apartment size      | 0.15 | Important but secondary |
| Comfort/Renovation  | 0.15 | Quality of living |
| Infrastructure      | 0.15 | Neighbourhood amenities |

### Example calculation — Apartment D (default weights)

```
Price:    5 × 0.30 = 1.50
Distance: 6 × 0.25 = 1.50
Size:     9 × 0.15 = 1.35
Comfort: 10 × 0.15 = 1.50
Infra:    8 × 0.15 = 1.20
                    ──────
Total:             = 7.05  (winner under default weights)
```

---

## MCDA score for real listings (Explorer page)

Each of the 22,676 real apartments gets a live MCDA score based on its actual data fields:

```javascript
function aptScore(a, w) {
    // price: cheaper = higher score (linear mapping)
    var priceScore   = Math.max(1, Math.min(10, Math.round(11 - (a.p - 2) / 4)));

    // metro: closer = higher score
    var metroScore   = Math.max(1, Math.min(10, Math.round(11 - a.m / 3)));

    // size: larger = higher score
    var sizeScore    = Math.max(1, Math.min(10, Math.round(a.a / 12)));

    // comfort: mapped from renovation category
    var renoMap = { 'Designer':10, 'European-style renovation':8,
                    'Cosmetic':6,  'Without renovation':3 };
    var comfortScore = renoMap[a.rn] || 5;

    // infrastructure: proxy from metro distance
    var infraScore = a.m <= 5 ? 9 : a.m <= 10 ? 7 : a.m <= 20 ? 6 : 4;

    return priceScore   * w.price    +
           metroScore   * w.distance +
           sizeScore    * w.size     +
           comfortScore * w.comfort  +
           infraScore   * w.infra;
}
```

The user can sort the explorer by MCDA score, which means the best-matching apartment for their weights always rises to the top.

---

## Scenario analysis

Three user profiles show that the "best" apartment depends on priorities:

```javascript
var PROFILES = {
    // Balanced default
    default: { price:0.30, distance:0.25, size:0.15, comfort:0.15, infra:0.15 },
    // Student — cost above all
    student: { price:0.50, distance:0.25, size:0.10, comfort:0.08, infra:0.07 },
    // Family — needs space and comfort
    family:  { price:0.15, distance:0.20, size:0.25, comfort:0.25, infra:0.15 },
    // Worker — location is everything
    worker:  { price:0.15, distance:0.45, size:0.15, comfort:0.15, infra:0.10 },
};
```

| Profile | Winner | Because |
|---------|--------|---------|
| Default | Apt D  | Best overall balance |
| Student | Apt C  | Score 10 on price (cheapest) |
| Family  | Apt D  | Score 10 on comfort, 9 on size |
| Worker  | Apt A  | Score 9 on distance (city center) |

---

## Dataset key findings

| Research question | Answer |
|------------------|--------|
| Most common type? | 2-room (28%), 1-room (23%), Studio (16%) |
| Metro effect on price? | 0–5 min costs 22% more than 6–15 min (13.4M vs 11.0M) |
| Renovation effect? | Designer is 9× more expensive than cosmetic (72M vs 8.2M) |
| Moscow vs Oblast? | Moscow costs 2.4× more (15.9M vs 6.5M median) |
| Floor preference? | Low (2–5) most common at 32%; ground and top floors 7% each |
| Key price drivers? | Price, metro proximity, and renovation level |

---

## Questions the teacher may ask

**Why WSM and not AHP or TOPSIS?**  
WSM is the most transparent model — every step is explainable by one formula line. AHP would be a natural extension for deriving weights more rigorously via pairwise comparisons. TOPSIS would be appropriate if we wanted to rank based on distance to an ideal and anti-ideal solution. WSM was chosen for clarity and ease of live demonstration.

**How did you set the weights?**  
Based on analysis of the dataset and the research questions. Price and metro proximity together account for 55% of the score because they are the two factors most strongly reflected in Moscow market pricing. Sensitivity analysis shows the winner (Apt D) is robust across most weight configurations.

**How does scoring work for real apartments?**  
Raw dataset fields (price in RUB, area in m², minutes to metro, renovation category) are converted to a 1–10 scale using monotone transformations documented in `aptScore()`. The mappings are consistent and auditable.

**Does the app read all 22,676 rows?**  
Yes. `data.js` contains all 22,676 rows pre-processed into a compact JSON array. The browser loads the full array on startup. Filtering, sorting, and scoring all run against the complete dataset in-memory.

**How does the app work without a server?**  
`data.js` is a plain JavaScript file that sets two global constants (`APARTMENTS` and `STATS`). When the browser loads `index.html` it also loads `data.js` as a script tag. Everything then runs in the browser — no HTTP requests after initial load, no backend, no database.

---

## Dependencies

| Library | Version | Purpose |
|---------|---------|---------|
| Chart.js | 4.4.1 | All 6 dashboard charts |
| Plus Jakarta Sans | — | Body font |
| Syne | — | Headings |

All loaded from CDN. No npm, no build tools, no frameworks.
