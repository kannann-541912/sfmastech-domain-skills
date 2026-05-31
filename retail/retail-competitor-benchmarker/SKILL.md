---
name: retail-competitor-benchmarker
description: Build a retail competitor benchmarking solution from existing Gold/Silver layer tables in DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT. Creates dynamic tables that aggregate competitor pricing data into retailer-level and brand-vs-retailer scorecards, a Cortex Search service for natural language competitor lookup, and a semantic view for Cortex Analyst. Use when asked to benchmark retailers, compare competitors, analyze competitive positioning, build a competitor scorecard, or assess pricing threats.
---

# Retail Competitor Benchmarker Skill

## Overview
This skill builds a production-grade retail competitor benchmarking solution from existing data in `DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT`. It produces:
1. A `COMPETITOR_BENCHMARK_360` dynamic table (one row per competitor-category with aggregated KPIs)
2. A `BRAND_VS_COMPETITOR_SCORECARD` dynamic table (brand × competitor head-to-head pricing analysis)
3. A `COMPETITOR_BENCHMARK_SEARCH` Cortex Search service for natural language competitor queries
4. A `RETAIL_COMPETITOR_BENCHMARK_ANALYTICS` semantic view for Cortex Analyst
5. Example benchmarking queries

## Source Tables

All tables reside in `DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT`:

| Table | Layer | Grain | Key Columns |
|-------|-------|-------|-------------|
| SILVER_COMPETITOR_PRICE_ENRICHED | Silver | One row per SKU × competitor price observation | SKU_ID, COMPETITOR_NAME |
| GOLD_SKU_PRICING_INTELLIGENCE | Gold | One row per active SKU with latest pricing | SKU_ID |
| GOLD_CATEGORY_PERFORMANCE_SUMMARY | Gold | One row per category (dynamic table) | CATEGORY, FEED_RUN_ID |
| GOLD_WEEKLY_BRAND_SELL_THROUGH | Gold | One row per brand × week | BRAND, WEEK_START_DATE |

## Key Relationships

- `SKU_ID` (VARCHAR) is the universal join key between competitor prices and our pricing intelligence
- `COMPETITOR_NAME` (VARCHAR) identifies the competing retailer
- `CATEGORY` (VARCHAR) segments by product category: AUDIO, GAMING, HOME_THEATER, LAPTOPS, SMARTPHONES
- `BRAND` (VARCHAR) identifies the manufacturer: Apple, Samsung, Sony, Dell, LG, Bose, JBL, Microsoft, Lenovo, Hisense
- `COMPETITIVE_POSITIONING_STATUS` classifies each SKU-competitor match: OVERPRICED, UNDERPRICED, PRICE_PARITY

## Schema Details

### SILVER_COMPETITOR_PRICE_ENRICHED
SILVER_RECORD_ID, BRONZE_SNAPSHOT_RECORD_ID, FEED_RUN_ID, SILVER_PROCESSED_AT, SKU_ID, PRODUCT_NAME, CATEGORY, COMPETITOR_NAME, COMPETITOR_PRICE, OUR_CURRENT_PRICE, SCRAPED_AT, PRICE_GAP_ABSOLUTE_USD, PRICE_GAP_PERCENTAGE, COMPETITIVE_POSITIONING_STATUS, COMPETITOR_HAS_PRODUCT_IN_STOCK, IS_SIGNIFICANT_PRICE_OPPORTUNITY, IS_STOCKOUT_OPPORTUNITY

Key facts:
- 480 rows across 131 unique competitors, 94 SKUs, 5 categories
- `PRICE_GAP_PERCENTAGE`: (our_price - competitor_price) / competitor_price × 100. Positive = we are more expensive.
- `COMPETITIVE_POSITIONING_STATUS`: OVERPRICED (gap > 5%), UNDERPRICED (gap < -5%), PRICE_PARITY (within ±5%)
- `IS_SIGNIFICANT_PRICE_OPPORTUNITY`: TRUE when we are >10% overpriced
- `IS_STOCKOUT_OPPORTUNITY`: TRUE when competitor is OOS and we have stock

### GOLD_SKU_PRICING_INTELLIGENCE
DASHBOARD_RECORD_ID, FEED_RUN_ID, GOLD_LAST_REFRESHED_AT, SKU_ID, PRODUCT_NAME, CATEGORY, SUBCATEGORY, BRAND, OUR_CURRENT_PRICE, OUR_COST_PRICE, OUR_MARGIN_PERCENTAGE, OUR_MARGIN_AMOUNT_USD, LOWEST_COMPETITOR_PRICE_FOUND, HIGHEST_COMPETITOR_PRICE_FOUND, NUMBER_OF_COMPETITORS_FOUND, PRIMARY_COMPETITOR_NAME, PRIMARY_COMPETITOR_PRICE, PRICE_GAP_VS_PRIMARY_COMPETITOR_PCT, OVERALL_COMPETITIVE_POSITIONING_STATUS, REQUIRES_IMMEDIATE_PRICE_REVIEW, CURRENT_STOCK_LEVEL, STOCK_AVAILABILITY_STATUS, ESTIMATED_DAYS_UNTIL_STOCKOUT, REQUIRES_REPLENISHMENT_ACTION, CORTEX_AI_RECOMMENDATION_SUMMARY, CORTEX_AI_RECOMMENDED_ACTION_TYPE, CORTEX_AI_RECOMMENDATION_GENERATED_AT

Key facts:
- 95 active SKUs across 5 categories
- Contains brand, margin, stock, and AI recommendation data per SKU
- `OVERALL_COMPETITIVE_POSITIONING_STATUS`: aggregated positioning across all competitors

---

## Step 1: Create COMPETITOR_BENCHMARK_360 Dynamic Table

This table aggregates competitor pricing data to the competitor-category level, providing a holistic view of each competitor's pricing behavior and threat level.

**Grain**: One row per (COMPETITOR_NAME, CATEGORY)

```sql
CREATE OR REPLACE DYNAMIC TABLE DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT.COMPETITOR_BENCHMARK_360
    COMMENT = 'Dynamic table. Competitor-level benchmarking KPIs by category. One row per competitor-category combination. Auto-refreshes from SILVER_COMPETITOR_PRICE_ENRICHED.'
    LAG = '30 minutes'
    REFRESH_MODE = AUTO
    INITIALIZE = ON_CREATE
    WAREHOUSE = RETAIL_CATEGORY_ANALYTICS_WH
AS
    SELECT
        MD5(comp.COMPETITOR_NAME || '|' || comp.CATEGORY) AS BENCHMARK_RECORD_ID,
        comp.COMPETITOR_NAME,
        comp.CATEGORY,
        COUNT(DISTINCT comp.SKU_ID) AS TOTAL_SKUS_MATCHED,
        ROUND(AVG(comp.PRICE_GAP_PERCENTAGE), 2) AS AVG_PRICE_GAP_PCT,
        ROUND(MEDIAN(comp.PRICE_GAP_PERCENTAGE), 2) AS MEDIAN_PRICE_GAP_PCT,
        ROUND(COUNT(CASE WHEN comp.COMPETITIVE_POSITIONING_STATUS = 'OVERPRICED' THEN 1 END) * 100.0 / NULLIF(COUNT(*), 0), 1) AS PCT_SKUS_WE_ARE_OVERPRICED,
        ROUND(COUNT(CASE WHEN comp.COMPETITIVE_POSITIONING_STATUS = 'UNDERPRICED' THEN 1 END) * 100.0 / NULLIF(COUNT(*), 0), 1) AS PCT_SKUS_WE_ARE_UNDERPRICED,
        ROUND(COUNT(CASE WHEN comp.COMPETITIVE_POSITIONING_STATUS = 'PRICE_PARITY' THEN 1 END) * 100.0 / NULLIF(COUNT(*), 0), 1) AS PCT_SKUS_AT_PARITY,
        ROUND(MIN(comp.COMPETITOR_PRICE), 2) AS MIN_COMPETITOR_PRICE,
        ROUND(MAX(comp.COMPETITOR_PRICE), 2) AS MAX_COMPETITOR_PRICE,
        ROUND(AVG(comp.COMPETITOR_PRICE), 2) AS AVG_COMPETITOR_PRICE,
        ROUND(AVG(comp.OUR_CURRENT_PRICE), 2) AS AVG_OUR_PRICE,
        ROUND(COUNT(CASE WHEN comp.COMPETITOR_HAS_PRODUCT_IN_STOCK = TRUE THEN 1 END) * 100.0 / NULLIF(COUNT(*), 0), 1) AS COMPETITOR_STOCK_AVAILABILITY_RATE,
        COUNT(CASE WHEN comp.IS_SIGNIFICANT_PRICE_OPPORTUNITY = TRUE THEN 1 END) AS PRICE_OPPORTUNITY_COUNT,
        COUNT(CASE WHEN comp.IS_STOCKOUT_OPPORTUNITY = TRUE THEN 1 END) AS STOCKOUT_OPPORTUNITY_COUNT,
        -- Composite threat score: price aggressiveness (40%) + category coverage breadth (30%) + stock availability (30%)
        ROUND(
            (COUNT(CASE WHEN comp.COMPETITIVE_POSITIONING_STATUS = 'OVERPRICED' THEN 1 END) * 100.0 / NULLIF(COUNT(*), 0)) * 0.4
            + (COUNT(DISTINCT comp.SKU_ID) * 100.0 / 94.0) * 0.3
            + (COUNT(CASE WHEN comp.COMPETITOR_HAS_PRODUCT_IN_STOCK = TRUE THEN 1 END) * 100.0 / NULLIF(COUNT(*), 0)) * 0.3
        , 1) AS COMPETITIVE_THREAT_SCORE
    FROM DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT.SILVER_COMPETITOR_PRICE_ENRICHED comp
    WHERE comp.COMPETITOR_NAME IS NOT NULL
    GROUP BY comp.COMPETITOR_NAME, comp.CATEGORY;
```

**Column Definitions:**

| Column | Description |
|--------|-------------|
| BENCHMARK_RECORD_ID | Deterministic unique key (MD5 of competitor + category) |
| COMPETITOR_NAME | Name of the competing retailer (e.g., Best Buy, Target, Walmart) |
| CATEGORY | Product category (AUDIO, GAMING, HOME_THEATER, LAPTOPS, SMARTPHONES) |
| TOTAL_SKUS_MATCHED | Number of our SKUs this competitor also carries |
| AVG_PRICE_GAP_PCT | Average % price gap. Positive = we are more expensive |
| MEDIAN_PRICE_GAP_PCT | Median % price gap (less sensitive to outliers) |
| PCT_SKUS_WE_ARE_OVERPRICED | % of SKUs where we lose on price |
| PCT_SKUS_WE_ARE_UNDERPRICED | % of SKUs where we win on price |
| PCT_SKUS_AT_PARITY | % of SKUs within ±5% of competitor |
| MIN/MAX/AVG_COMPETITOR_PRICE | Price range and average for this competitor |
| AVG_OUR_PRICE | Our average price on matched SKUs |
| COMPETITOR_STOCK_AVAILABILITY_RATE | % of products this competitor has in stock |
| PRICE_OPPORTUNITY_COUNT | SKUs where they significantly overcharge vs us |
| STOCKOUT_OPPORTUNITY_COUNT | SKUs where they're OOS but we have stock |
| COMPETITIVE_THREAT_SCORE | 0-100 composite score. Higher = bigger threat |

---

## Step 2: Create BRAND_VS_COMPETITOR_SCORECARD Dynamic Table

This table provides head-to-head brand-level analysis, answering "How are we doing with Samsung products vs Best Buy?"

**Grain**: One row per (BRAND, COMPETITOR_NAME, CATEGORY)

```sql
CREATE OR REPLACE DYNAMIC TABLE DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT.BRAND_VS_COMPETITOR_SCORECARD
    COMMENT = 'Dynamic table. Brand vs competitor head-to-head pricing scorecard. One row per brand-competitor-category combination.'
    LAG = '30 minutes'
    REFRESH_MODE = AUTO
    INITIALIZE = ON_CREATE
    WAREHOUSE = RETAIL_CATEGORY_ANALYTICS_WH
AS
    SELECT
        MD5(sku.BRAND || '|' || comp.COMPETITOR_NAME || '|' || comp.CATEGORY) AS SCORECARD_RECORD_ID,
        sku.BRAND,
        comp.COMPETITOR_NAME,
        comp.CATEGORY,
        COUNT(DISTINCT comp.SKU_ID) AS SKUS_IN_COMMON,
        ROUND(AVG(comp.OUR_CURRENT_PRICE), 2) AS AVG_OUR_PRICE,
        ROUND(AVG(comp.COMPETITOR_PRICE), 2) AS AVG_COMPETITOR_PRICE,
        ROUND(AVG(comp.PRICE_GAP_PERCENTAGE), 2) AS AVG_PRICE_GAP_PCT,
        ROUND(AVG(sku.OUR_MARGIN_PERCENTAGE), 2) AS OUR_AVG_MARGIN_PCT,
        COUNT(CASE WHEN comp.COMPETITIVE_POSITIONING_STATUS = 'UNDERPRICED' THEN 1 END) AS PRICE_WIN_COUNT,
        COUNT(CASE WHEN comp.COMPETITIVE_POSITIONING_STATUS = 'OVERPRICED' THEN 1 END) AS PRICE_LOSS_COUNT,
        COUNT(CASE WHEN comp.COMPETITIVE_POSITIONING_STATUS = 'PRICE_PARITY' THEN 1 END) AS PARITY_COUNT,
        ROUND(
            COUNT(CASE WHEN comp.COMPETITIVE_POSITIONING_STATUS = 'UNDERPRICED' THEN 1 END) * 100.0 
            / NULLIF(COUNT(*), 0)
        , 1) AS WIN_RATE_PCT,
        ROUND(
            SUM(CASE 
                WHEN comp.PRICE_GAP_ABSOLUTE_USD > 0 THEN comp.PRICE_GAP_ABSOLUTE_USD * COALESCE(sku.OUR_MARGIN_PERCENTAGE / 100.0, 0.2)
                ELSE 0 
            END)
        , 2) AS TOTAL_MARGIN_AT_RISK_USD,
        CASE
            WHEN COUNT(CASE WHEN comp.COMPETITIVE_POSITIONING_STATUS = 'UNDERPRICED' THEN 1 END) * 100.0 / NULLIF(COUNT(*), 0) >= 60 THEN 'WINNING'
            WHEN COUNT(CASE WHEN comp.COMPETITIVE_POSITIONING_STATUS = 'OVERPRICED' THEN 1 END) * 100.0 / NULLIF(COUNT(*), 0) >= 60 THEN 'LOSING'
            ELSE 'COMPETITIVE'
        END AS HEAD_TO_HEAD_VERDICT
    FROM DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT.SILVER_COMPETITOR_PRICE_ENRICHED comp
    INNER JOIN DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT.GOLD_SKU_PRICING_INTELLIGENCE sku
        ON comp.SKU_ID = sku.SKU_ID
    WHERE comp.COMPETITOR_NAME IS NOT NULL
      AND sku.BRAND IS NOT NULL
    GROUP BY sku.BRAND, comp.COMPETITOR_NAME, comp.CATEGORY;
```

**Column Definitions:**

| Column | Description |
|--------|-------------|
| SCORECARD_RECORD_ID | Deterministic unique key |
| BRAND | Our brand/manufacturer (Apple, Samsung, Sony, etc.) |
| COMPETITOR_NAME | The competing retailer |
| CATEGORY | Product category |
| SKUS_IN_COMMON | Overlapping SKU count |
| AVG_OUR_PRICE | Our avg price on shared SKUs |
| AVG_COMPETITOR_PRICE | Their avg price on shared SKUs |
| AVG_PRICE_GAP_PCT | Avg % gap. Positive = we are more expensive |
| OUR_AVG_MARGIN_PCT | Our margin on overlapping SKUs |
| PRICE_WIN_COUNT | SKUs where we are cheaper |
| PRICE_LOSS_COUNT | SKUs where they are cheaper |
| PARITY_COUNT | SKUs at price parity |
| WIN_RATE_PCT | (wins / total) × 100 |
| TOTAL_MARGIN_AT_RISK_USD | Margin we'd lose if we price-matched on losing SKUs |
| HEAD_TO_HEAD_VERDICT | WINNING (≥60% win rate), LOSING (≥60% loss rate), or COMPETITIVE |

---

## Step 3: Create Cortex Search Service

The Cortex Search service enables natural language queries like "Which competitor undercuts us most in Audio?" or "Find retailers with stockout opportunities in Smartphones."

### Step 3a: Create the search base view

```sql
CREATE OR REPLACE VIEW DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT.COMPETITOR_BENCHMARK_SEARCH_BASE AS
SELECT
    MD5(COMPETITOR_NAME || '|' || CATEGORY) AS BENCHMARK_RECORD_ID,
    COMPETITOR_NAME,
    CATEGORY,
    COUNT(DISTINCT SKU_ID) AS TOTAL_SKUS_MATCHED,
    ROUND(AVG(PRICE_GAP_PERCENTAGE), 2) AS AVG_PRICE_GAP_PCT,
    ROUND(COUNT(CASE WHEN COMPETITIVE_POSITIONING_STATUS = 'OVERPRICED' THEN 1 END) * 100.0 / NULLIF(COUNT(*), 0), 1) AS PCT_WE_ARE_OVERPRICED,
    CONCAT(
        'Competitor: ', COMPETITOR_NAME, '. ',
        'Category: ', CATEGORY, '. ',
        'Matches ', COUNT(DISTINCT SKU_ID)::VARCHAR, ' of our SKUs. ',
        'Average price gap: ', ROUND(AVG(PRICE_GAP_PERCENTAGE), 1)::VARCHAR, '% (positive means we are more expensive). ',
        'We are overpriced on ', ROUND(COUNT(CASE WHEN COMPETITIVE_POSITIONING_STATUS = 'OVERPRICED' THEN 1 END) * 100.0 / NULLIF(COUNT(*), 0), 0)::VARCHAR, '% of matched SKUs. ',
        'We are underpriced on ', ROUND(COUNT(CASE WHEN COMPETITIVE_POSITIONING_STATUS = 'UNDERPRICED' THEN 1 END) * 100.0 / NULLIF(COUNT(*), 0), 0)::VARCHAR, '% of matched SKUs. ',
        'Price parity on ', ROUND(COUNT(CASE WHEN COMPETITIVE_POSITIONING_STATUS = 'PRICE_PARITY' THEN 1 END) * 100.0 / NULLIF(COUNT(*), 0), 0)::VARCHAR, '% of matched SKUs. ',
        'Significant price opportunities: ', COUNT(CASE WHEN IS_SIGNIFICANT_PRICE_OPPORTUNITY = TRUE THEN 1 END)::VARCHAR, '. ',
        'Stockout opportunities (they are OOS, we have inventory): ', COUNT(CASE WHEN IS_STOCKOUT_OPPORTUNITY = TRUE THEN 1 END)::VARCHAR, '. ',
        CASE 
            WHEN COUNT(CASE WHEN COMPETITIVE_POSITIONING_STATUS = 'OVERPRICED' THEN 1 END) * 100.0 / NULLIF(COUNT(*), 0) >= 60 THEN 'This competitor is a HIGH THREAT - they undercut us on most products in this category.'
            WHEN COUNT(CASE WHEN COMPETITIVE_POSITIONING_STATUS = 'OVERPRICED' THEN 1 END) * 100.0 / NULLIF(COUNT(*), 0) >= 40 THEN 'This competitor is a MODERATE THREAT - they undercut us on many products in this category.'
            ELSE 'This competitor is a LOW THREAT - we are competitively positioned against them in this category.'
        END
    ) AS SEARCH_NARRATIVE
FROM DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT.SILVER_COMPETITOR_PRICE_ENRICHED
WHERE COMPETITOR_NAME IS NOT NULL
GROUP BY COMPETITOR_NAME, CATEGORY;
```

### Step 3b: Create the Cortex Search service

```sql
CREATE OR REPLACE CORTEX SEARCH SERVICE DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT.COMPETITOR_BENCHMARK_SEARCH
    ON SEARCH_NARRATIVE
    ATTRIBUTES COMPETITOR_NAME, CATEGORY, TOTAL_SKUS_MATCHED, AVG_PRICE_GAP_PCT, PCT_WE_ARE_OVERPRICED
    WAREHOUSE = RETAIL_CATEGORY_ANALYTICS_WH
    TARGET_LAG = '30 minutes'
    COMMENT = 'Cortex Search service for natural language competitor benchmarking queries. Search for competitors by pricing behavior, category, threat level, or opportunities.'
AS (
    SELECT
        BENCHMARK_RECORD_ID,
        COMPETITOR_NAME,
        CATEGORY,
        TOTAL_SKUS_MATCHED,
        AVG_PRICE_GAP_PCT,
        PCT_WE_ARE_OVERPRICED,
        SEARCH_NARRATIVE
    FROM DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT.COMPETITOR_BENCHMARK_SEARCH_BASE
);
```

---

## Step 4: Create Semantic View

The semantic view enables Cortex Analyst to answer natural language questions about competitor benchmarking.

```sql
CREATE OR REPLACE SEMANTIC VIEW DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT.RETAIL_COMPETITOR_BENCHMARK_ANALYTICS

  TABLES (
    competitor_benchmarks AS DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT.COMPETITOR_BENCHMARK_360
      PRIMARY KEY (BENCHMARK_RECORD_ID)
      COMMENT = 'Competitor-level benchmarking KPIs. One row per competitor-category pair with pricing gaps, threat scores, and opportunity counts.',
    brand_scorecards AS DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT.BRAND_VS_COMPETITOR_SCORECARD
      PRIMARY KEY (SCORECARD_RECORD_ID)
      COMMENT = 'Brand vs competitor head-to-head scorecard. One row per brand-competitor-category triple with win/loss rates and margin at risk.'
  )

  RELATIONSHIPS (
    scorecard_to_benchmark AS
      brand_scorecards (COMPETITOR_NAME, CATEGORY) REFERENCES competitor_benchmarks (COMPETITOR_NAME, CATEGORY)
  )

  FACTS (
    competitor_benchmarks.total_skus_matched AS TOTAL_SKUS_MATCHED
      COMMENT = 'Number of our SKUs this competitor also carries in this category',
    competitor_benchmarks.avg_price_gap_pct AS AVG_PRICE_GAP_PCT
      COMMENT = 'Average price gap percentage vs this competitor. Positive means we are more expensive.',
    competitor_benchmarks.median_price_gap_pct AS MEDIAN_PRICE_GAP_PCT
      COMMENT = 'Median price gap percentage vs this competitor.',
    competitor_benchmarks.pct_overpriced AS PCT_SKUS_WE_ARE_OVERPRICED
      COMMENT = 'Percentage of matched SKUs where we are overpriced vs this competitor',
    competitor_benchmarks.pct_underpriced AS PCT_SKUS_WE_ARE_UNDERPRICED
      COMMENT = 'Percentage of matched SKUs where we are cheaper than this competitor',
    competitor_benchmarks.pct_parity AS PCT_SKUS_AT_PARITY
      COMMENT = 'Percentage of matched SKUs at price parity with this competitor',
    competitor_benchmarks.avg_competitor_price AS AVG_COMPETITOR_PRICE
      COMMENT = 'Average price charged by this competitor in USD',
    competitor_benchmarks.avg_our_price AS AVG_OUR_PRICE
      COMMENT = 'Our average price for matched SKUs in USD',
    competitor_benchmarks.stock_availability AS COMPETITOR_STOCK_AVAILABILITY_RATE
      COMMENT = 'Percentage of matched products this competitor has in stock',
    competitor_benchmarks.price_opportunities AS PRICE_OPPORTUNITY_COUNT
      COMMENT = 'Count of SKUs with significant price opportunity',
    competitor_benchmarks.stockout_opportunities AS STOCKOUT_OPPORTUNITY_COUNT
      COMMENT = 'Count of SKUs where competitor is OOS but we have inventory',
    competitor_benchmarks.threat_score AS COMPETITIVE_THREAT_SCORE
      COMMENT = 'Composite threat score 0-100. Higher = bigger competitive threat.',
    brand_scorecards.skus_in_common AS SKUS_IN_COMMON
      COMMENT = 'Number of overlapping SKUs for this brand-competitor-category',
    brand_scorecards.scorecard_avg_our_price AS AVG_OUR_PRICE
      COMMENT = 'Our average price for overlapping SKUs in USD',
    brand_scorecards.scorecard_avg_competitor_price AS AVG_COMPETITOR_PRICE
      COMMENT = 'Competitor average price for overlapping SKUs in USD',
    brand_scorecards.scorecard_avg_price_gap AS AVG_PRICE_GAP_PCT
      COMMENT = 'Average price gap percent for brand vs competitor. Positive = we are more expensive.',
    brand_scorecards.our_margin AS OUR_AVG_MARGIN_PCT
      COMMENT = 'Our average margin percentage on overlapping SKUs',
    brand_scorecards.wins AS PRICE_WIN_COUNT
      COMMENT = 'Number of SKUs where we are cheaper',
    brand_scorecards.losses AS PRICE_LOSS_COUNT
      COMMENT = 'Number of SKUs where competitor is cheaper',
    brand_scorecards.parity AS PARITY_COUNT
      COMMENT = 'Number of SKUs at price parity',
    brand_scorecards.win_rate AS WIN_RATE_PCT
      COMMENT = 'Percentage of SKUs where we win on price',
    brand_scorecards.margin_at_risk AS TOTAL_MARGIN_AT_RISK_USD
      COMMENT = 'Total margin at risk if we matched competitor prices on losing SKUs'
  )

  DIMENSIONS (
    competitor_benchmarks.competitor_name AS COMPETITOR_NAME
      WITH SYNONYMS = ('retailer', 'seller', 'competing store', 'rival')
      COMMENT = 'Name of the competing retailer (e.g., Best Buy, Target, Walmart, Samsung)',
    competitor_benchmarks.category AS CATEGORY
      WITH SYNONYMS = ('product category', 'department', 'segment')
      COMMENT = 'Product category: AUDIO, GAMING, HOME_THEATER, LAPTOPS, SMARTPHONES',
    brand_scorecards.brand AS BRAND
      WITH SYNONYMS = ('manufacturer', 'maker', 'vendor')
      COMMENT = 'Brand/manufacturer name: Apple, Samsung, Sony, Dell, LG, Bose, JBL, Microsoft, Lenovo, Hisense',
    brand_scorecards.verdict AS HEAD_TO_HEAD_VERDICT
      WITH SYNONYMS = ('result', 'outcome', 'competitive status')
      COMMENT = 'Overall head-to-head verdict: WINNING (win rate >= 60%), LOSING (loss rate >= 60%), or COMPETITIVE'
  )

  METRICS (
    competitor_benchmarks.avg_threat AS AVG(competitor_benchmarks.threat_score)
      COMMENT = 'Average competitive threat score across competitors',
    brand_scorecards.avg_win_rate AS AVG(brand_scorecards.win_rate)
      COMMENT = 'Average price win rate across brand-competitor matchups'
  )

  COMMENT = 'Retail competitor benchmarking analytics. Compare retailers head-to-head on pricing, coverage, and competitive positioning by category and brand. Use competitor_benchmarks for retailer-level KPIs. Use brand_scorecards for brand vs retailer head-to-head analysis.';
```

---

## Step 5: Example Queries

### Benchmarking queries against COMPETITOR_BENCHMARK_360

```sql
-- Top 10 most threatening competitors overall
SELECT COMPETITOR_NAME, CATEGORY, COMPETITIVE_THREAT_SCORE, 
       TOTAL_SKUS_MATCHED, AVG_PRICE_GAP_PCT, PCT_SKUS_WE_ARE_OVERPRICED
FROM DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT.COMPETITOR_BENCHMARK_360
ORDER BY COMPETITIVE_THREAT_SCORE DESC
LIMIT 10;

-- Competitors where we are overpriced on more than 50% of matched SKUs
SELECT COMPETITOR_NAME, CATEGORY, PCT_SKUS_WE_ARE_OVERPRICED, 
       AVG_PRICE_GAP_PCT, TOTAL_SKUS_MATCHED
FROM DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT.COMPETITOR_BENCHMARK_360
WHERE PCT_SKUS_WE_ARE_OVERPRICED > 50
ORDER BY PCT_SKUS_WE_ARE_OVERPRICED DESC;

-- Category-level competitive landscape: how many competitors by threat level
SELECT CATEGORY,
       COUNT(CASE WHEN COMPETITIVE_THREAT_SCORE >= 70 THEN 1 END) AS HIGH_THREAT_COMPETITORS,
       COUNT(CASE WHEN COMPETITIVE_THREAT_SCORE BETWEEN 40 AND 69 THEN 1 END) AS MODERATE_THREAT_COMPETITORS,
       COUNT(CASE WHEN COMPETITIVE_THREAT_SCORE < 40 THEN 1 END) AS LOW_THREAT_COMPETITORS
FROM DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT.COMPETITOR_BENCHMARK_360
GROUP BY CATEGORY
ORDER BY HIGH_THREAT_COMPETITORS DESC;

-- Stockout opportunities: competitors OOS where we have stock
SELECT COMPETITOR_NAME, CATEGORY, STOCKOUT_OPPORTUNITY_COUNT, TOTAL_SKUS_MATCHED
FROM DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT.COMPETITOR_BENCHMARK_360
WHERE STOCKOUT_OPPORTUNITY_COUNT > 0
ORDER BY STOCKOUT_OPPORTUNITY_COUNT DESC;
```

### Head-to-head brand queries against BRAND_VS_COMPETITOR_SCORECARD

```sql
-- How are we doing vs Best Buy across all brands?
SELECT BRAND, CATEGORY, SKUS_IN_COMMON, WIN_RATE_PCT, 
       AVG_PRICE_GAP_PCT, TOTAL_MARGIN_AT_RISK_USD, HEAD_TO_HEAD_VERDICT
FROM DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT.BRAND_VS_COMPETITOR_SCORECARD
WHERE COMPETITOR_NAME = 'Best Buy'
ORDER BY TOTAL_MARGIN_AT_RISK_USD DESC;

-- Which brand-competitor matchups are we LOSING?
SELECT BRAND, COMPETITOR_NAME, CATEGORY, WIN_RATE_PCT, 
       PRICE_LOSS_COUNT, TOTAL_MARGIN_AT_RISK_USD
FROM DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT.BRAND_VS_COMPETITOR_SCORECARD
WHERE HEAD_TO_HEAD_VERDICT = 'LOSING'
ORDER BY TOTAL_MARGIN_AT_RISK_USD DESC;

-- Compare two retailers head-to-head for Samsung products
SELECT COMPETITOR_NAME, CATEGORY, SKUS_IN_COMMON, AVG_PRICE_GAP_PCT, 
       WIN_RATE_PCT, HEAD_TO_HEAD_VERDICT
FROM DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT.BRAND_VS_COMPETITOR_SCORECARD
WHERE BRAND = 'Samsung'
  AND COMPETITOR_NAME IN ('Best Buy', 'Target')
ORDER BY COMPETITOR_NAME, CATEGORY;

-- Total margin at risk by brand (summed across all competitors)
SELECT BRAND, 
       SUM(TOTAL_MARGIN_AT_RISK_USD) AS TOTAL_MARGIN_AT_RISK,
       ROUND(AVG(WIN_RATE_PCT), 1) AS AVG_WIN_RATE
FROM DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT.BRAND_VS_COMPETITOR_SCORECARD
GROUP BY BRAND
ORDER BY TOTAL_MARGIN_AT_RISK DESC;
```

### Cortex Search queries

```sql
-- Natural language search for competitor insights
SELECT PARSE_JSON(
    SNOWFLAKE.CORTEX.SEARCH_PREVIEW(
        'DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT.COMPETITOR_BENCHMARK_SEARCH',
        '{
            "query": "which competitor is cheapest in gaming",
            "columns": ["COMPETITOR_NAME", "CATEGORY", "AVG_PRICE_GAP_PCT", "PCT_WE_ARE_OVERPRICED"],
            "limit": 5
        }'
    )
) AS results;

-- Search for stockout opportunities
SELECT PARSE_JSON(
    SNOWFLAKE.CORTEX.SEARCH_PREVIEW(
        'DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT.COMPETITOR_BENCHMARK_SEARCH',
        '{
            "query": "retailers out of stock where we have inventory in smartphones",
            "columns": ["COMPETITOR_NAME", "CATEGORY", "TOTAL_SKUS_MATCHED", "AVG_PRICE_GAP_PCT"],
            "limit": 5
        }'
    )
) AS results;
```

---

## Execution Instructions

When this skill is invoked:

1. **Set context**: Run `USE DATABASE DEMO_DEV; USE SCHEMA RETAIL_CATEGORY_ANALYTICS_AGENT; USE WAREHOUSE RETAIL_CATEGORY_ANALYTICS_WH;` to ensure all objects are created in `DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT`.

2. **Create the COMPETITOR_BENCHMARK_360 dynamic table** using the SQL in Step 1. Execute it and verify with `SELECT COUNT(*) FROM DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT.COMPETITOR_BENCHMARK_360;` (expected: ~130-200 rows depending on competitor-category combinations).

3. **Create the BRAND_VS_COMPETITOR_SCORECARD dynamic table** using the SQL in Step 2. Verify with `SELECT COUNT(*) FROM DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT.BRAND_VS_COMPETITOR_SCORECARD;`.

4. **Create the Cortex Search base view** using Step 3a SQL, then **create the Cortex Search service** using Step 3b SQL. Wait for initialization and test with a sample search query.

5. **Create the semantic view** using the SQL in Step 4. All objects must exist before this step.

6. **Validate** by running the example queries in Step 5. Confirm threat scores are computed, verdicts are assigned, and search returns relevant results.

7. **Report results** including row counts for both dynamic tables, sample top-5 threats, and confirmation that Cortex Search and the semantic view are operational.

**Target Schema**: All objects (COMPETITOR_BENCHMARK_360, BRAND_VS_COMPETITOR_SCORECARD, COMPETITOR_BENCHMARK_SEARCH_BASE, COMPETITOR_BENCHMARK_SEARCH, RETAIL_COMPETITOR_BENCHMARK_ANALYTICS) MUST be created in `DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT`.

## Real-World Value

This solution:
- Enables category managers to instantly benchmark against any competitor without manual spreadsheet analysis
- Provides brand-level competitive intelligence (e.g., "We're losing to Best Buy on Sony products in Gaming")
- Quantifies margin at risk from competitive pricing pressure
- Identifies stockout opportunities to capture competitor's lost demand
- Reduces competitive analysis cycle time from days of manual work to real-time queries
- Supports natural language access via Cortex Search and Cortex Analyst for non-technical users
- Auto-refreshes every 30 minutes as new competitor price data flows in
