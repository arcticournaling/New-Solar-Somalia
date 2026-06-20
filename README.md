# New Solar

New Solar is a lightweight MVP website for a commercial solar calculator focused on businesses in Mogadishu.

## Tech stack

- Astro for pages and routing
- Svelte for the calculator component only
- Plain CSS
- No authentication
- No maps
- Local browser storage for saved estimates

## Pages

- Home: `/`
- Calculator: `/calculator`
- Methodology: `/methodology`

## Getting started

Install dependencies:

```bash
npm install
```

Start the development server:

```bash
npm run dev
```

Build for production:

```bash
npm run build
```

Preview the production build:

```bash
npm run preview
```

## Calculator assumptions

- Average peak sun hours in Mogadishu: 5.7 hours/day
- System efficiency factor: 0.75
- Panel power rating: 550W
- Estimated roof area per panel: 2.6 square meters
- Installed solar cost range: $1,000-$1,600 per kW
- Battery cost estimate: $300-$500 per kWh
- Battery recommendation: 30% of daily use when backup is selected
- Example battery module size: 5 kWh

## Calculation notes

The main calculation logic lives in `src/components/SolarCalculator.svelte`.

Version 1 uses these formulas:

```text
daily_kwh = monthly_kwh / 30
coverage_decimal = coverage_percentage / 100
target_daily_solar = daily_kwh * coverage_decimal
system_size_kw = target_daily_solar / (peak_sun_hours * efficiency_factor)
panel_count = ceil((system_size_kw * 1000) / panel_watts)
roof_space_required = panel_count * panel_area
max_panels_by_roof = floor(roof_area / panel_area)
max_system_size_kw = (max_panels_by_roof * panel_watts) / 1000
effective_system_size_kw = roof-limited size when roof space is constrained
monthly_solar_production = effective_system_size_kw * peak_sun_hours * 30 * efficiency_factor
monthly_grid_savings = min(monthly_solar_production, monthly_kwh) * electricity_cost_per_kwh
diesel_savings = 0 for none, monthly_diesel_cost * 0.25 for occasional, monthly_diesel_cost * 0.6 for daily
total_monthly_savings = monthly_grid_savings + diesel_savings
solar_cost_low = effective_system_size_kw * 1000
solar_cost_high = effective_system_size_kw * 1600
battery_storage_kwh = daily_kwh * 0.3
battery_module_count = ceil(battery_storage_kwh / 5)
battery_cost_low = battery_storage_kwh * 300
battery_cost_high = battery_storage_kwh * 500
installation_soft_cost_low = solar_cost_low * 0.15
installation_soft_cost_high = solar_cost_high * 0.25
estimated_cost_low = solar_cost_low + battery_cost_low + installation_soft_cost_low
estimated_cost_high = solar_cost_high + battery_cost_high + installation_soft_cost_high
simple_payback_years_low = estimated_cost_low / (total_monthly_savings * 12)
simple_payback_years_high = estimated_cost_high / (total_monthly_savings * 12)
```

## Local storage today, Supabase later

Submissions are currently saved to `localStorage` under the key `newSolarSubmissions`.

To add Supabase later, replace the `saveSubmission` function in `src/components/SolarCalculator.svelte` with an API call or Supabase client insert. The saved object already separates `answers`, `results`, and `createdAt` so it can map cleanly to a future database table.

## Disclaimer

New Solar estimates are for early planning only. They are not a final engineering design, structural review, electrical design, or installation quote.
