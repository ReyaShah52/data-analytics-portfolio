# Toothpaste Carbon Footprint & Price Dashboard

## Project Overview

This project builds an end-to-end Power BI dashboard comparing six toothpaste brands across retail price and estimated carbon footprint. The entire data pipeline — price collection, carbon parameter extraction, and lifecycle formula calculation — runs automatically inside Power Query using live API calls. No data is entered manually.

I leveraged SerpApi's Google Shopping API to pull product and price data directly from Google. To support the carbon footprint analysis, I used Google NotebookLM to source lifecycle assessment research papers and connected the Anthropic Claude API to Power BI using JSON code to read those documents and output specific carbon metrics for each product.

The dashboard introduces an Efficiency Score (0–100) that combines carbon footprint and price into a single comparable metric, weighted 70% toward carbon. A scatter plot surfaces the value-efficiency landscape across all six brands, and individual gauge visuals show each product's penalty score — where a higher score means less efficient.

## Business Problem

Carbon footprint data for consumer products is buried in lifecycle assessment research PDFs. Retail prices change frequently and vary by retailer. No single tool brings these two dimensions together for direct brand comparison.

Key questions this dashboard answers:
- Which toothpaste brand has the lowest carbon footprint per unit?
- How does retail price compare across brands?
- Which brand offers the best efficiency — lowest carbon relative to price?
- Is there a correlation between product price and environmental impact?

## Dashboard Preview

![Toothpaste Carbon Footprint Dashboard](dashboard_1.png)

## Tools Used

- Power BI / Power Query M
- SerpApi — Google Shopping API
- Anthropic Claude API
- Google NotebookLM
- DAX
- Excel

## Products Analyzed

| Product | Avg Price | Avg Carbon (g CO2e) | Efficiency Score |
|---|---|---|---|
| Himalaya Complete Care 5.29 oz | $10.76 | 1,092 | 93 / 100 |
| Crest Pro Health Clean Mint 4.3 oz | $8.72 | 863 | 53 / 100 |
| Aquafresh Cavity Protection 5.6 oz | $7.89 | 892 | 51 / 100 |
| Sensodyne Pronamel Intensive Enamel Repair 3.4 oz | $12.03 | 630 | 44 / 100 |
| Colgate Total Whitening 4.8 oz | $6.91 | 823 | 37 / 100 |
| Tom's Complete Care Fresh Mint 4 oz | $9.76 | 517 | 17 / 100 |

## Methodology

### Step 1 — Research: Google NotebookLM
Used Google NotebookLM to locate published lifecycle assessment (LCA) research PDFs covering toothpaste carbon footprints. Key documents were consolidated into a single reference PDF referenced by the Anthropic API connection inside Power Query.

### Step 2 — Live Price Pull: SerpApi (Power Query M)
A reusable M function calls the SerpApi Google Shopping endpoint once per product and returns the top result — title, price, retailer, and link. A separate Brand Avg Price query groups results by brand using List.Average to produce one averaged price row per brand.

### Step 3 — Carbon Footprint Extraction: Anthropic Claude API
A Power Query function reads the research PDF using Pdf.Tables, extracts the text, and sends a structured prompt to Claude (claude-sonnet-4-6) for each product. Claude returns a JSON object with seven lifecycle parameters. Power Query parses the response and computes the carbon formula inline.

Carbon formula: carbon = 1000 * (
(Net_Mass_kg  * Formula_EF)        -- ingredient production
(Pkg_Total_kg * Pkg_Life_EF)       -- packaging lifecycle
(Net_Mass_kg  * Dist_km * Mode_EF) -- transport emissions
Soil_Credit                        -- soil drawdown offset
)

### Step 4 — DAX Data Modeling
Two calculated columns were added to the Carbon Footprint Data table (one row per brand) to power the visuals. All normalization logic was consolidated into a single Efficiency Score column to avoid filter context errors from the multi-row Products table.

```dax
Avg Price =
LOOKUPVALUE(
    'Brand Avg Price'[Avg Price],
    'Brand Avg Price'[Brand],
    'Carbon Footprint Data'[Brand]
)
```

```dax
Efficiency Score =
VAR MinP = 6.9081   VAR MaxP = 12.0286
VAR MinC = 516.984  VAR MaxC = 1092.5
VAR PriceScore  = DIVIDE(MaxP - 'Carbon Footprint Data'[Avg Price],  MaxP - MinP, 0)
VAR CarbonScore = DIVIDE(MaxC - 'Carbon Footprint Data'[Total Grams CO2e], MaxC - MinC, 0)
VAR RawScore    = PriceScore * 0.3 + CarbonScore * 0.7
RETURN ROUND((1 - RawScore) * 100, 0)
```

### Step 5 — Dashboard Visuals
- **Scatter plot** — X axis: Avg Price, Y axis: Total Grams CO2e, bubble size: Efficiency Score, color: Brand
- **Gauge visuals** — Six individual gauges filtered by Brand at the visual level, Min = 0, Max = 100, Value = Efficiency Score

## Key Results

- **Lowest carbon footprint:** Tom's Complete Care (~517g CO2e) — nearly half the carbon of the next closest brand
- **Best efficiency:** Tom's Complete Care (score 17) — strong carbon advantage outweighs its mid-range price
- **Worst performer:** Himalaya Complete Care (score 93) — highest carbon combined with a mid-high price
- **No price-carbon correlation:** Sensodyne is most expensive yet has the second lowest carbon. Colgate is cheapest but mid-range carbon. Price is not a reliable indicator of environmental impact — carbon footprint is driven by ingredient sourcing, packaging materials, and supply chain geography, none of which are reflected in retail pricing

## Skills Demonstrated

- API Integration (SerpApi + Anthropic Claude)
- Prompt Engineering
- Power Query / M
- Carbon Footprint Modeling
- DAX Calculated Columns
- Data Modeling & Filter Context Troubleshooting
- Dashboard Design

## Project Outcome

The dashboard demonstrates that Power BI can function as a full integration layer by embedding live API calls and AI inference directly into the data refresh pipeline. The Efficiency Score reveals Tom's Complete Care as the most efficient brand and confirms that price and carbon footprint have no meaningful correlation — meaning consumers need a tool like this to make informed environmental choices, since shelf price alone provides no useful signal.
