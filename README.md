# The State of World Happiness

This project explores where happiness is concentrated, what factors explain differences between countries, and how national well-being changes over time. The dashboard is built around the World Happiness data (Gallup World Poll, 2012–2024) and uses the additive decomposition of the happiness score to compare component factors.

## How to Run
1. Clone this repository: `git clone https://github.com/rvoak/ECS272-Winter2026-FinalProject.git`
2. Change into the cloned directory: `cd ECS272-Winter2026-FinalProject`
3. Install dependencies: `npm install`
4. Start the dev server: `npm run dev`
5. Open the local URL printed in the terminal.

## Dashboard Overview
The dashboard provides two main views:
- **Map view**: global choropleth with country selection and factor comparison panel.
- **Movers view**: top gainers/decliners list with a detailed trends panel.

## Visuals and Interactions
### Choropleth Map
- **What it shows**: country-level happiness scores (0–10) for a selected year, or score deltas between two years.
- **Interactions**:
  - Year selector and play button (Score mode) to animate through all years.
  - Delta mode with two year selectors to compare change across years.
  - Hover shows country name, score, and rank (or delta values in Delta mode).
  - Click a country to add it to the Factor Comparison Panel (up to five). Click again to remove.
  - Selected countries are outlined in black for clarity.

### Factor Comparison Panel
- **What it shows**: stacked bars per selected country; bar length is total score, segment widths are factor contributions.
- **Interactions**:
  - Hover segments to see factor name and value.
  - Close button clears the selection.

### Movers List
- **What it shows**: top 10 gains and top 10 declines in rank between selected years.
- **Interactions**:
  - Start and End year selectors.
  - Click a country bar to open the trends panel.
  - Hover bars to see year:rank (or year:score when Score mode is enabled).

### Movers Detail Panel
- **What it shows**: two line charts for the selected country.
  - **Top**: score or rank trend over time.
  - **Bottom**: factor trends (one line per factor), using the same colors as the Factor Comparison Panel.
- **Interactions**:
  - Country dropdown to switch the trend view.
  - Hover any point to see year and value.
  - Close button to dismiss the panel.

# Acknowledgements
The template for this dashboard was used from the one provided in the [class homeworks](https://github.com/VIDITeaching/ECS272-Winter2026/tree/main/Homework2). AI tools (specifically, OpenAI Codex) were used for two tasks: generating this readme file based on the code, and formatting / prettifying the javascript code itself. 
