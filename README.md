# Marketing Intelligence Dashboard

> **Marketing intelligence dashboard supporting data-driven marketing decisions using social media analytics and competitive benchmarking across Facebook, Instagram, and TikTok.**

![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Pandas](https://img.shields.io/badge/Pandas-150458?style=for-the-badge&logo=pandas&logoColor=white)
![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![Jupyter](https://img.shields.io/badge/Jupyter-F37626?style=for-the-badge&logo=jupyter&logoColor=white)
![Facebook](https://img.shields.io/badge/Facebook-1877F2?style=for-the-badge&logo=facebook&logoColor=white)
![Instagram](https://img.shields.io/badge/Instagram-E4405F?style=for-the-badge&logo=instagram&logoColor=white)
![TikTok](https://img.shields.io/badge/TikTok-000000?style=for-the-badge&logo=tiktok&logoColor=white)

---

## Project Overview

Most marketing teams measure social media performance by total followers or raw likes — numbers that are easy to read but often misleading. This project was built to go deeper: to turn twelve months of raw, unstructured social media data from three platforms into a structured analytical model that answers real questions about content efficiency, not just content volume.

The project follows a complete analytics pipeline — from data collection through ETL, data modeling, DAX, and interactive reporting — and produces a **9-page Power BI dashboard** covering performance exploration, opportunity detection, and data-backed recommendations per platform.

**Important framing:** the benchmark brand (CR) is used here not as a direct competitor to "beat," but as an external reference point to better understand audience behavior and identify content patterns that resonate — a learning tool for the marketing team, not a scorecard.


---

## Tech Stack

| Layer | Tool |
|---|---|
| Data Collection | Apify (Facebook, Instagram, TikTok scrapers) |
| Data Engineering | Python 3.13, pandas |
| Text Processing | regex (Arabic + English NLP cleaning) |
| Data Modeling | Power BI — Star Schema (Power Query) |
| Analytical Layer | DAX — 60+ custom measures |
| Visualization | Power BI — 9-page interactive report |
| Version Control | Git & GitHub |

---

## End-to-End Pipeline

```
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│   Social Platforms          Python ETL           Power BI              │
│   (Public Pages)            Pipeline             Model & Report        │
│                                                                        │
│  ┌──────────────┐        ┌───────────────┐      ┌───────────────────┐  │
│  │   Facebook   │─────▶ │  FB Cleaning   │──▶  │  Fact_Facebook    │  │
│  │   Instagram  │─────▶ │  IG Cleaning   │──▶  │  Fact_Instagram   │  │
│  │   TikTok     │─────▶ │  TT Cleaning   │──▶  │  Fact_TikTok      │  │
│  └──────────────┘        └────────────────┘     │  Dim_Calendar     │  │
│         ▲                      │                │  Dim_Brand        │  │
│         │                      │                └────────┬──────────┘  │
│      Apify                 Fact CSVs                   │               │
│     Scrapers              (UTF-8 BOM)              DAX Layer           │
│                                                        │               │
│                                                  9-Page Report         │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Repository Structure

```
Marketing-Intelligence-Dashboard/
│
├── Python/
│   ├── facebook_data_preprocessing.ipynb     # Facebook ETL — GM + CR (multi-batch)
│   ├── instagram_data_preprocessing.ipynb    # Instagram ETL — GM + CR
│   └── tiktok_data_preprocessing.ipynb       # TikTok ETL — GM + CR
│
├── Screenshots/
│   ├── Home page.png
│   ├── Performance Exploration _ Facebook.png
│   ├── Performance Exploration_Instagram.png
│   ├── Performance Exploration_TikTok.png
│   ├── Performance Improvement_Facebook_GMC.png
│   └── Executive Recommendations.png
│
└── README.md
```

---

## Data Engineering (Python)

All three Apify exports arrive as nested JSON with different schemas per platform. The Python layer normalizes every source file into a clean, analysis-ready fact table. The same core pipeline was applied consistently across all six source files (GM + CR × 3 platforms):

### What each notebook does

**Schema selection**
Only the fields relevant to analysis were retained — engagement metrics, timestamps, content type flags, and caption text. Everything else was dropped at the ETL stage to keep the model lean.

| Platform | Key fields retained |
|---|---|
| Facebook | `postId`, `text`, `isVideo`, `paidPartnership`, `likes`, `comments`, `shares`, `viewsCount`, 7 reaction types |
| Instagram | `id`, `caption`, `productType`, `likesCount`, `commentsCount`, `videoViewCount`, `videoPlayCount`, `videoDuration`, `hashtags`, `isPinned` |
| TikTok | `text`, `diggCount`, `commentCount`, `shareCount`, `playCount`, `collectCount`, `videoMeta.duration`, music metadata, `createTimeISO` |

**Missing value handling**
Text fields → empty string; numeric engagement fields → `0`; booleans → `False`. This prevents aggregation errors downstream without imputing artificial values.

**Bilingual text cleaning (Arabic + English)**
A regex-based pipeline was written specifically to handle mixed Arabic/English captions — the dominant writing style in Egyptian retail content:
```python
# Preserve Arabic Unicode block explicitly alongside English
df["cleaned_text"] = df["text"].str.replace(r"[^\w\s\u0600-\u06FF]", " ", regex=True)
```
The pipeline removes URLs, `@mentions`, emojis, and special characters while keeping both scripts intact. A naive English-only cleaner would have deleted most of the Arabic caption content.

**Content type derivation (Facebook)**
```python
df["ContentType"] = df["isVideo"].map({True: "Video", False: "Image"})
```
Converted the raw boolean into a readable label used later in the content-mix analysis visual.

**Surrogate key generation**
Every post gets a unique `PostKey` encoding platform, brand, and source batch, so records from multiple scraping runs merge without collision:
```
FB_GOM_1_<postId>   → Facebook, GM, batch 1
IG_CARR_1_<id>      → Instagram, CR, batch 1
TT_GOM_<index>      → TikTok, GM
```

**Brand & Platform tagging**
Every record is explicitly tagged at ETL (`Brand`, `Platform`) rather than inferred from file names later — this keeps the star schema clean and the slicers reliable.

**Export**
All output CSVs use `encoding="utf-8-sig"` (UTF-8 with BOM) to preserve Arabic text integrity when loaded into Power BI.

---

## Data Model — Star Schema

Rather than keeping a single flat table, the cleaned CSVs were loaded into a proper star schema. This is what makes cross-platform, cross-brand slicing from a single unified date filter and brand filter possible.

```
                    ┌─────────────────┐
                    │   Dim_Calendar  │
                    │─────────────────│
                    │ Date (PK)       │
                    │ DayName         │
                    │ DayNumber       │
                    │ MonthName       │
                    │ MonthNumber     │
                    │ Quarter         │
                    │ Year            │
                    └────────┬────────┘
                             │ 1
              ┌──────────────┼───────────────────────┐
             *│             *│                      *│
    ┌─────────────────┐  ┌──────────────────┐  ┌────────────────┐
    │  Fact_Facebook  │  │  Fact_Instagram  │  │  Fact_TikTok   │
    │─────────────────│  │──────────────────│  │────────────────│
    │ PostKey (PK)    │  │ PostKey (PK)     │  │ PostID (PK)    │
    │ Brand (FK)      │  │ Brand (FK)       │  │ Brand (FK)     │
    │ PostDate (FK)   │  │ PostDate (FK)    │  │ PostDate (FK)  │
    │ Category        │  │ Category         │  │ Category       │
    │ Caption         │  │ Caption          │  │ Caption        │
    │ IsVideo         │  │ PostType         │  │ IsOriginalSound│
    │ PaidPartnership │  │ IsPinned         │  │ MusicName      │
    │ Likes           │  │ Likes            │  │ Likes          │
    │ Comments        │  │ Comments         │  │ Comments       │
    │ Shares          │  │ VideoPlays       │  │ Shares         │
    │ VideoViews      │  │ VideoViews       │  │ Collects       │
    │ Reaction*       │  │ VideoDuration    │  │ VideoViews     │
    │ Hour            │  │ Hashtags         │  │ VideoDuration  │
    └────────┬────────┘  │ Hour             │  │ Hour           │
             │           └────────┬─────────┘  └───────┬────────┘
             └─────────────────── ┼ ───────────────────┘
                                  │ *
                         ┌────────┴────────┐
                         │   Dim_Brand     │
                         │─────────────────│
                         │ Brand (PK)      │
                         └─────────────────┘
```

**Design decisions worth noting:**
- `Dim_Calendar` and `Dim_Brand` are **conformed dimensions** shared across all three fact tables — one date slicer and one brand filter drive the entire report.
- Every fact table carries a **`Category`** field enabling like-for-like content comparison (Ramadan GM vs. Ramadan CR, not Ramadan vs. an arbitrary post).
- An **`Hour`** field is derived on every fact table to power the engagement heatmaps on the Improvement Opportunities pages.

---

## DAX Measures Library

All measures are centralized in an `AllMeasure` table and organized into per-platform folders using a consistent `Platform - Brand - Metric` naming convention, so any reviewer can navigate the measure list without opening a single visual.

### Facebook (`FB` folder — 12 measures)

```
FB - Angry vs Likes Rate          FB - GOM Total Followers
FB - Avg Engagement per Post      FB - GOM Total Posts
FB - CRF Engagement per 1k        FB - Total Likes
FB - CRF Total Followers          FB - Total Posts
FB - CRF Total Posts              FB - Video Engagement per View
FB - GOM Engagement per 1k        Winning Icon
```

### Instagram (`IG` folder — 12 measures)

```
IG - Winning Icon                 IG - GOM Engagement per 1k
IG - Avg Engagement per Post      IG - GOM Total Followers
IG - CRF Engagement per 1k        IG - GOM Total Posts
IG - CRF Total Followers          IG - Total Engagement
IG - CRF Total Posts              IG - Total Likes
                                   IG - Total Posts
                                   IG - Video Engagement per View
```

### TikTok (`tiktok` folder — 12 measures)

```
TT - Avg Engagement per Post      TT - GOM Engagement per 1k
TT - Collects Rate                TT - GOM Total Followers
TT - CRF Engagement per 1k        TT - Total Collects
TT - CRF Total Followers          TT - Total Engagement
                                   TT - Total Likes
                                   TT - Total Posts
                                   TT - Video Engagement per View
                                   TT - Winning Icon
```

### Measures worth highlighting

**`[Platform] - GOM/CRF Engagement per 1k`** (all platforms)
Engagement normalized per 1,000 followers, per brand. This is the single most important metric in the model — it's what reveals that GM's content was actually *more efficient* per follower than CR's on two of the three platforms, despite CR winning every raw/absolute number. Without this measure, the report would have drawn the wrong conclusion.

**`FB - Angry vs Likes Rate`**
Ratio of negative-sentiment reactions (Angry) to positive ones (Like) — a lightweight proxy for content backlash risk that a plain "total reactions" figure would completely mask.

**`TT - Collects Rate`**
TikTok saves (Collects) as a share of views — used as a proxy for genuinely save-worthy content versus content that's merely watched once and scrolled past. GM's save rate (0.10%) was consistently higher than CR's (0.03%) across the analysis period.

**`TT - Video Engagement per View`** (and equivalents)
Normalizes video performance by actual views rather than follower count, making video content comparable across accounts of very different sizes.

**`Winning Icon`** (FB + IG + TT)
A dynamic conditional measure that drives the ▲ / ▼ indicators in the competitive category tables, so the "who's winning this category" read is instant rather than requiring a mental calculation.

---

## Dashboard — Sample Pages

> *Brand names and logos have been blurred in all screenshots. The full report contains more pages than shown here.*

### Home Page
![Home Page](Screenshots/Home%20page.png)

### Performance Exploration — Facebook
![Facebook Performance](Screenshots/Performance%20Exploration%20_%20Facebook.png)

### Performance Exploration — Instagram
![Instagram Performance](Screenshots/Performance%20Exploration_Instagram.png)

### Performance Exploration — TikTok
![TikTok Performance](Screenshots/Performance%20Exploration_TikTok.png)

### Improvement Opportunities — Facebook
![Facebook Opportunities](Screenshots/Performance%20Improvement_Facebook_GMC.png)

### Executive Recommendations
![Executive Recommendations](Screenshots/Executive%20Recommendations.png)

---

## Key Analytical Findings

These observations emerged from the data during analysis and are documented here to demonstrate the analytical reasoning behind the dashboard design — not as advice directed at any company.

**1. Follower count is a misleading proxy for content quality.**
On Facebook and TikTok, GM's engagement-per-1,000-followers consistently outperformed CR's, despite CR having a 5–13× larger follower base on those platforms. Every absolute metric told the opposite story; the normalized metric told the real one. This finding shaped the core KPI choice for the entire report.

**2. Instagram required a methodological correction before it could be used for benchmarking.**
Several of CR's top-performing Instagram categories showed an extreme statistical pattern — very high average engagement concentrated in a very small number of posts (e.g. one post averaging several thousand interactions, while 150 posts in the same category averaged far less). That signature is consistent with influencer/blogger amplification rather than organic brand content. Those figures were flagged and excluded from any "replicate the competitor" conclusion — illustrating why a metric needs to be interrogated, not just reported.

**3. Seasonal and occasion-based content was the strongest consistent performance driver.**
Across both brands and multiple platforms, content tied to Ramadan, Eid, and back-to-school seasons achieved higher average engagement than always-on categories — frequently with *fewer* posts. This pattern held across both Facebook and TikTok independently, which makes it more reliable than a single-platform observation.

**4. High posting volume can mask declining engagement efficiency.**
The largest category by post count ("offers & discounts") showed a downward engagement trend for CR on two platforms despite — or because of — its sheer volume. This was a useful saturation signal that a simple "post more of what works" approach would have missed entirely.

**5. Posting-time heatmaps needed outlier correction before they were actionable.**
A small number of individual posts that happened to go viral skewed the naive "best time to post" read significantly. Those single-cell spikes were identified and flagged rather than taken at face value — the actionable scheduling insight came from the *consistent* pattern across the heatmap, not the peak cells.

---

## Skills Demonstrated

| Skill Area | What was applied |
|---|---|
| Data Sourcing | Configuring and running Apify actors for multi-platform, multi-batch scraping |
| Python / pandas | JSON normalization, null handling, multi-batch deduplication, surrogate key design |
| Arabic NLP | Bilingual regex text cleaning that preserves Arabic Unicode while stripping noise |
| Data Modeling | Star-schema design with conformed dimensions across 3 fact tables |
| DAX | 60+ measures: normalization, rates, ratios, conditional formatting logic, time-intelligence |
| Power BI | Multi-page RTL-compatible report (Arabic language), heatmap scheduling analysis, trend forecasting visual |
| Analytical Judgment | Identifying and correcting for statistical bias (influencer-driven outliers) before drawing conclusions |

---

## About This Project

An independent portfolio project built over ~1 month to demonstrate end-to-end data analytics skills. Not affiliated with or commissioned by any company referenced in the data. All brand names are anonymized (GM / CR). Data was collected from public social media pages via Apify — no private or user-level data was accessed.

## 👤 Author

**Mahmoud Hamdi** — Data Analyst

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0A66C2?style=flat&logo=linkedin)](https://www.linkedin.com/in/mahmoud-hamdi-analyst) [![Email](https://img.shields.io/badge/Email-Contact-EA4335?style=flat&logo=gmail)](mailto:mahmoudhamdiwm@gmail.com)

---
