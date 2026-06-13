# Leadership Through Transition — Website Build Package
## Complete Technical Architecture Specification

**Version:** 1.0  
**Date:** June 2025  
**Stack:** Next.js 14 (App Router) + MDX + PostgreSQL  
**Document Authority:** This spec supersedes any conflicting structural descriptions in earlier project documents. Where source documents disagree, the canonical authority order is: (1) Book Entries, (2) Cross-Book Wisdom Framework, (3) Library Architecture Document, (4) Master Catalog.

---

## Document Revision Notes

Before any implementation begins, three coherence issues discovered during spec preparation must be resolved editorially. They are flagged here so the development team does not encode inconsistencies into the database schema.

| Issue | Document Conflict | Resolution Required |
|---|---|---|
| Node tier mismatch | *Leadership and Self-Deception* has 4 wisdom themes but is classified as Standard Node (3-theme tier) in the CrossBook Framework Node Map | Book Entry classification (4 themes) is authoritative. Promote to High-Density Node (4 themes). Update CrossBook Framework Node Map. |
| Tier conflict: *Start With Why* | Library Architecture lists as Supporting tier; Book Entry lists as Core / Foundational Canon | Book Entry is authoritative. The Library Architecture document predates the Book Entries. Treat as Core / Foundational Canon throughout the database. |
| Tier conflict: *The Infinite Game* | Library Architecture lists as Supporting; CrossBook Framework Node Map shows Standard Node (3 themes); Book Entry records 4 wisdom themes | Book Entry is authoritative. Reclassify as High-Density Node (4 themes). Update CrossBook Framework Node Map and Library Architecture. |

These three corrections should be made in the relevant documents before the database is seeded. The Master Catalog (v1.0, generated June 2025) already reflects the correct Book Entry classifications for all three.

---

## Part I — Information Architecture

### 1.1 Site Purpose

LTT is a leadership knowledge platform — not a book summary service. Every architectural decision must reflect that distinction. The platform helps military leaders navigate separation and transition through the wisdom of carefully selected books, editorial commentary written from lived military experience, and cross-book synthesis that no individual book can provide.

The three questions every page must be able to answer: What did the author teach? Why does it matter for leadership? How does it apply to major life transitions?

### 1.2 Primary Audience

**Primary:** Military officers and senior enlisted leaders approaching or navigating retirement and separation. Secondary: Active-duty military leaders, government leaders, first responders, civilian executives.

The voice and experience model for all content: a senior naval officer who has led people through real adversity, read widely across philosophy and organizational science, and has something honest and specific to say about what transition actually requires.

### 1.3 Content Inventory

| Content Type | Current Count | Planned (40-book collection) |
|---|---|---|
| Book Entries | 15 (complete) | 40 |
| Theme Intelligence Pages | 8 active themes | 15 (all wisdom themes) |
| Reading Pathway Guides | 6 complete | 10 |
| Wisdom Briefs | 1 | ~10 (one per major theme cluster) |
| Catalog / Metadata | Machine-readable JSON + XLSX | Database-driven |

### 1.4 Content Hierarchy

```
Platform
├── Library (all books)
│   ├── Book Entry (per book)
│   │   ├── Opening Hook
│   │   ├── Book Analysis
│   │   ├── Leadership Transition Commentary
│   │   ├── Practical Applications
│   │   ├── Reflection Questions
│   │   ├── Cross-Book Wisdom Connections
│   │   └── Closing Reflection
│   └── [Related Books sidebar]
│
├── Themes (15 wisdom themes)
│   └── Theme Intelligence Page (per theme)
│       ├── Theme Definition
│       ├── Primary Books
│       ├── Convergences
│       ├── Divergences
│       ├── Leadership Application
│       ├── Transition Application
│       └── Gateway Books
│
├── Pathways (10 reading pathways)
│   └── Pathway Guide (per pathway)
│       ├── Who This Is For
│       ├── Pathway Overview
│       ├── Reading Sequence (7–8 books)
│       ├── Accumulated Lessons
│       └── Closing Reflection
│
├── Wisdom Briefs (synthesis documents)
│   └── Brief (per theme cluster)
│       ├── Convergences
│       ├── Disagreements
│       ├── Gaps
│       ├── Practical Implications
│       └── Reading Map
│
└── Discover (AI-assisted entry)
    ├── Reader Assessment
    ├── Personalized Pathway
    └── AI Wisdom Assistant
```

### 1.5 Entry Points

Four reader profiles, each needing a distinct entry point:

| Profile | Entry Point | First Destination |
|---|---|---|
| "I need to understand myself before I transition" | Identity / self-discovery | Identity Beyond Rank pathway or Frankl entry |
| "I need to prepare for a civilian career" | Practical / career | First 90 Days entry or Building a Second Career pathway |
| "I want to be a better leader right now" | Leadership | Five Dysfunctions or Leaders Eat Last entry |
| "I want to understand why and what comes next" | Purpose / meaning | Start With Why or Finding Purpose After Service pathway |

---

## Part II — Database Schema

PostgreSQL. All text fields storing LTT editorial content are stored as markdown strings. MDX rendering happens at the application layer, not in the database.

### 2.1 Core Tables

```sql
-- ─────────────────────────────────────────────────────────────────
-- BOOKS
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE books (
  id                  SERIAL PRIMARY KEY,
  slug                VARCHAR(120) UNIQUE NOT NULL,       -- url-safe: "mans-search-for-meaning"
  catalog_id          VARCHAR(12) UNIQUE,                  -- "LTT-001"
  title               VARCHAR(200) NOT NULL,
  subtitle            VARCHAR(300),
  author              VARCHAR(200) NOT NULL,
  year                VARCHAR(20) NOT NULL,                -- "1946" or "~170–180 AD"
  tagline             TEXT,                                -- one-sentence description from entry
  library_tier        VARCHAR(20) NOT NULL                 -- 'Core' | 'Supporting' | 'Advanced' | 'Specialized'
                      CHECK (library_tier IN ('Core','Supporting','Advanced','Specialized')),
  node_tier           VARCHAR(30) NOT NULL,                -- 'Critical Node' | 'High-Density Node' | 'Standard Node' | 'Pathway Node'
  node_theme_count    SMALLINT NOT NULL DEFAULT 0,
  entry_version       VARCHAR(10),                         -- "2.0"
  entry_date          VARCHAR(30),                         -- "June 2025"
  gold_standard       BOOLEAN NOT NULL DEFAULT FALSE,
  entry_status        VARCHAR(20) NOT NULL DEFAULT 'complete'
                      CHECK (entry_status IN ('complete','draft','planned')),
  created_at          TIMESTAMPTZ DEFAULT NOW(),
  updated_at          TIMESTAMPTZ DEFAULT NOW()
);

-- ─────────────────────────────────────────────────────────────────
-- BOOK CONTENT SECTIONS
-- Each section maps to one structural section of the Book Entry Blueprint
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE book_sections (
  id                  SERIAL PRIMARY KEY,
  book_id             INTEGER NOT NULL REFERENCES books(id) ON DELETE CASCADE,
  section_key         VARCHAR(50) NOT NULL,
  -- Valid values: 'opening_hook' | 'book_analysis' | 'leadership_transition_commentary'
  --               'practical_applications' | 'reflection_questions'
  --               'cross_book_wisdom' | 'closing_reflection'
  content_mdx         TEXT NOT NULL,
  word_count          INTEGER,
  updated_at          TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE (book_id, section_key)
);

-- ─────────────────────────────────────────────────────────────────
-- CANON STATUS (many-to-many: a book can have multiple canon designations)
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE canon_designations (
  id                  SERIAL PRIMARY KEY,
  name                VARCHAR(80) UNIQUE NOT NULL
  -- e.g. 'Foundational Canon' | 'Philosophy & Meaning Canon' | 'Military Leadership Canon'
  --      'Core Leadership Canon' | 'Core Transition Canon' | 'Sinek Trilogy Completion'
);

CREATE TABLE book_canon (
  book_id             INTEGER NOT NULL REFERENCES books(id) ON DELETE CASCADE,
  canon_id            INTEGER NOT NULL REFERENCES canon_designations(id) ON DELETE CASCADE,
  PRIMARY KEY (book_id, canon_id)
);

-- ─────────────────────────────────────────────────────────────────
-- TAXONOMY: CATEGORIES, LEADERSHIP THEMES, TRANSITION THEMES
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE categories (
  id                  SERIAL PRIMARY KEY,
  name                VARCHAR(80) UNIQUE NOT NULL
  -- From Taxonomy & Content Standards approved list
  -- e.g. 'Philosophy' | 'Psychology' | 'Military Leadership' | 'Business' ...
);

CREATE TABLE leadership_themes (
  id                  SERIAL PRIMARY KEY,
  name                VARCHAR(80) UNIQUE NOT NULL
  -- Approved list: 'Resilience' | 'Self-Awareness' | 'Character' | 'Accountability' ...
);

CREATE TABLE transition_themes (
  id                  SERIAL PRIMARY KEY,
  name                VARCHAR(80) UNIQUE NOT NULL
  -- Approved list: 'Identity' | 'Purpose' | 'Meaning' | 'Belonging' | 'Legacy' ...
);

CREATE TABLE book_categories      (book_id INT REFERENCES books(id) ON DELETE CASCADE, category_id INT REFERENCES categories(id), PRIMARY KEY (book_id, category_id));
CREATE TABLE book_leadership_tags (book_id INT REFERENCES books(id) ON DELETE CASCADE, theme_id INT REFERENCES leadership_themes(id), PRIMARY KEY (book_id, theme_id));
CREATE TABLE book_transition_tags (book_id INT REFERENCES books(id) ON DELETE CASCADE, theme_id INT REFERENCES transition_themes(id), PRIMARY KEY (book_id, theme_id));

-- ─────────────────────────────────────────────────────────────────
-- WISDOM THEMES (the 15 Cross-Book Wisdom Engine themes)
-- These are distinct from leadership_themes and transition_themes
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE wisdom_themes (
  id                  SERIAL PRIMARY KEY,
  slug                VARCHAR(60) UNIQUE NOT NULL,         -- 'identity' | 'purpose' | 'meaning' ...
  name                VARCHAR(60) UNIQUE NOT NULL,
  governing_question  VARCHAR(200),                        -- "Who are you when the rank is gone?"
  theme_definition    TEXT,                                -- from Theme Intelligence Pages
  leadership_application TEXT,
  transition_application TEXT,
  theme_evolution_notes TEXT,
  display_order       SMALLINT DEFAULT 0
);

-- Primary wisdom themes per book (from entry metadata)
CREATE TABLE book_wisdom_themes (
  book_id             INTEGER NOT NULL REFERENCES books(id) ON DELETE CASCADE,
  wisdom_theme_id     INTEGER NOT NULL REFERENCES wisdom_themes(id) ON DELETE CASCADE,
  is_primary          BOOLEAN DEFAULT TRUE,                -- primary = listed in entry metadata
  PRIMARY KEY (book_id, wisdom_theme_id)
);

-- ─────────────────────────────────────────────────────────────────
-- CROSS-BOOK WISDOM CONNECTIONS (the Cross-Book Wisdom Engine)
-- ─────────────────────────────────────────────────────────────────
CREATE TYPE relationship_type AS ENUM (
  'Convergence', 'Extension', 'Contrast', 'DiagnosticPair',
  'Application', 'HistoricalEcho', 'TraditionBridge'
);

CREATE TABLE wisdom_connections (
  id                  SERIAL PRIMARY KEY,
  source_book_id      INTEGER NOT NULL REFERENCES books(id) ON DELETE CASCADE,
  target_book_id      INTEGER NOT NULL REFERENCES books(id) ON DELETE CASCADE,
  relationship_type   relationship_type NOT NULL,
  wisdom_theme_ids    INTEGER[],                           -- array of wisdom_theme ids
  connection_label    VARCHAR(200),                        -- e.g. "Historical Echo · Character + Resilience"
  connection_body     TEXT NOT NULL,                       -- full prose connection (75–150 words per spec)
  appears_in_entry    INTEGER REFERENCES books(id),        -- which book entry this connection lives in
  display_order       SMALLINT DEFAULT 0,
  CONSTRAINT no_self_connection CHECK (source_book_id != target_book_id)
);

-- ─────────────────────────────────────────────────────────────────
-- RELATED BOOKS (ordered list per entry, distinct from wisdom connections)
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE book_related (
  book_id             INTEGER NOT NULL REFERENCES books(id) ON DELETE CASCADE,
  related_book_id     INTEGER NOT NULL REFERENCES books(id) ON DELETE CASCADE,
  display_order       SMALLINT NOT NULL DEFAULT 0,         -- 1–5
  connection_rationale TEXT,                               -- why this connection matters for this reader
  PRIMARY KEY (book_id, related_book_id),
  CONSTRAINT no_self_relation CHECK (book_id != related_book_id)
);

-- ─────────────────────────────────────────────────────────────────
-- SEARCH KEYWORDS
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE book_keywords (
  id                  SERIAL PRIMARY KEY,
  book_id             INTEGER NOT NULL REFERENCES books(id) ON DELETE CASCADE,
  keyword             VARCHAR(80) NOT NULL,
  UNIQUE (book_id, keyword)
);

-- ─────────────────────────────────────────────────────────────────
-- READING PATHWAYS
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE pathways (
  id                  SERIAL PRIMARY KEY,
  slug                VARCHAR(80) UNIQUE NOT NULL,          -- 'identity-beyond-rank'
  name                VARCHAR(120) NOT NULL,                 -- 'Identity Beyond Rank'
  tagline             VARCHAR(200),                          -- 'For leaders navigating the loss...'
  reader_profile      TEXT,
  primary_need        TEXT,
  pathway_arc         TEXT,
  book_count          SMALLINT,
  reading_weeks_min   SMALLINT,
  reading_weeks_max   SMALLINT,
  intersects_with     VARCHAR[],                             -- slugs of intersecting pathways
  intro_prose         TEXT,                                  -- 'Reading this pathway' section
  closing_reflection  TEXT,
  display_order       SMALLINT DEFAULT 0
);

CREATE TABLE pathway_books (
  pathway_id          INTEGER NOT NULL REFERENCES pathways(id) ON DELETE CASCADE,
  book_id             INTEGER NOT NULL REFERENCES books(id) ON DELETE CASCADE,
  step_number         SMALLINT NOT NULL,
  why_it_belongs      TEXT NOT NULL,
  what_it_builds      TEXT NOT NULL,
  PRIMARY KEY (pathway_id, book_id)
);

-- Accumulated lessons (one row per step in each pathway)
CREATE TABLE pathway_lessons (
  id                  SERIAL PRIMARY KEY,
  pathway_id          INTEGER NOT NULL REFERENCES pathways(id) ON DELETE CASCADE,
  after_step          SMALLINT NOT NULL,                     -- "After Books 1–3"
  after_label         VARCHAR(120),                          -- "After Books 1–3 — + Frankl"
  accumulated_lesson  TEXT NOT NULL
);

-- ─────────────────────────────────────────────────────────────────
-- THEME INTELLIGENCE PAGES
-- ─────────────────────────────────────────────────────────────────
-- Core theme data is in wisdom_themes table
-- This table stores the additional intelligence page sections

CREATE TABLE theme_pages (
  id                  SERIAL PRIMARY KEY,
  wisdom_theme_id     INTEGER NOT NULL UNIQUE REFERENCES wisdom_themes(id) ON DELETE CASCADE,
  convergences_json   JSONB,
  -- Array of: { books: string[], relationship_type: string, label: string, body: string }
  divergences_json    JSONB,
  -- Array of: { label: string, body: string }
  gateway_books_json  JSONB
  -- Array of: { book_slug: string, rationale: string }
);

-- ─────────────────────────────────────────────────────────────────
-- WISDOM BRIEFS
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE wisdom_briefs (
  id                  SERIAL PRIMARY KEY,
  slug                VARCHAR(100) UNIQUE NOT NULL,
  title               VARCHAR(200) NOT NULL,
  subtitle            TEXT,
  version             VARCHAR(10),
  published_date      VARCHAR(30),
  books_synthesized   INTEGER[],                             -- array of book_ids
  content_mdx         TEXT NOT NULL,                         -- full brief in MDX
  reading_map_json    JSONB                                  -- structured reading sequence
);

-- ─────────────────────────────────────────────────────────────────
-- SUBJECT AREAS (Library Architecture clusters)
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE subject_areas (
  id                  SERIAL PRIMARY KEY,
  name                VARCHAR(100) UNIQUE NOT NULL,
  description         TEXT
);

CREATE TABLE book_subject_areas (
  book_id             INTEGER NOT NULL REFERENCES books(id) ON DELETE CASCADE,
  subject_area_id     INTEGER NOT NULL REFERENCES subject_areas(id) ON DELETE CASCADE,
  PRIMARY KEY (book_id, subject_area_id)
);

-- ─────────────────────────────────────────────────────────────────
-- KEY LESSONS (structured list, queryable separately from full content)
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE book_key_lessons (
  id                  SERIAL PRIMARY KEY,
  book_id             INTEGER NOT NULL REFERENCES books(id) ON DELETE CASCADE,
  lesson_number       SMALLINT NOT NULL,
  lesson_text         TEXT NOT NULL,
  PRIMARY KEY (book_id, lesson_number)
);

-- ─────────────────────────────────────────────────────────────────
-- FULL-TEXT SEARCH SUPPORT
-- ─────────────────────────────────────────────────────────────────
-- tsvector column for full-text search across book content
ALTER TABLE books ADD COLUMN search_vector TSVECTOR;

CREATE INDEX idx_books_search ON books USING GIN(search_vector);
CREATE INDEX idx_books_slug ON books(slug);
CREATE INDEX idx_books_tier ON books(library_tier);
CREATE INDEX idx_books_node ON books(node_tier);
CREATE INDEX idx_wisdom_connections_source ON wisdom_connections(source_book_id);
CREATE INDEX idx_wisdom_connections_target ON wisdom_connections(target_book_id);
CREATE INDEX idx_pathway_books_pathway ON pathway_books(pathway_id);
CREATE INDEX idx_book_wisdom_themes ON book_wisdom_themes(wisdom_theme_id);
```

### 2.2 Key Constraints and Rules

- Every book in `entry_status = 'complete'` must have all 7 section keys present in `book_sections`.
- `node_theme_count` must equal the count of rows in `book_wisdom_themes` where `is_primary = TRUE` for that book.
- Node tier must be consistent with theme count: Critical Node ≥ 5, High-Density Node = 4, Standard Node = 3, Pathway Node ≤ 2.
- Every `wisdom_connection` must have at least one `wisdom_theme_id` in its array.
- `book_related` rows must have `display_order` values 1–5 with no gaps per book.

### 2.3 Seed Data Order

1. `wisdom_themes` (15 themes)
2. `categories`, `leadership_themes`, `transition_themes`
3. `canon_designations`
4. `books` (40 records, status = 'complete' | 'planned')
5. `book_sections` (for all complete entries)
6. All junction tables
7. `pathways` + `pathway_books` + `pathway_lessons`
8. `theme_pages`
9. `wisdom_briefs`
10. Rebuild `search_vector` column

---

## Part III — Content Model

### 3.1 Book Entry MDX Structure

Each of the 7 content sections is stored separately in `book_sections.content_mdx` and rendered independently. This allows the application to show individual sections, inject sidebar content between sections, and build the Wisdom Engine overlay without reformatting the full entry.

**Section key → Blueprint section mapping:**

| section_key | Blueprint Section | Spec Word Count |
|---|---|---|
| `opening_hook` | Section 1 | 200–300 words |
| `book_analysis` | Section 2 | 600–900 words |
| `leadership_transition_commentary` | Section 3 | 600–900 words |
| `practical_applications` | Section 4 | 350–500 words |
| `reflection_questions` | Section 5 | 300–400 words (4 questions) |
| `cross_book_wisdom` | Section 6 | 600–900 words (4 connections) |
| `closing_reflection` | Section 7 | 150–250 words |

**MDX conventions for book entry content:**

```mdx
{/* opening_hook — no headings, pure prose */}
Somewhere between the decision to leave...

{/* book_analysis — two sub-headings allowed */}
## The Central Argument
...
## Key Frameworks and Ideas
...

{/* reflection_questions — structured table */}
| Category | Question |
|---|---|
| **Identity & Self-Awareness** | When you describe... |

{/* cross_book_wisdom — uses LTT-specific components */}
<WisdomConnection
  targetBook="meditations"
  relationshipType="HistoricalEcho"
  themes={["Character", "Resilience"]}
  label="Historical Echo · Character + Resilience"
>
Aurelius and Frankl never encountered each other...
</WisdomConnection>
```

### 3.2 MDX Component Library (LTT-Specific)

These components are defined in `src/components/ltt/` and used within MDX content:

```tsx
// Cross-book wisdom connection block
<WisdomConnection
  targetBook={string}           // book slug
  relationshipType={RelType}    // enum
  themes={string[]}             // wisdom theme names
  label={string}                // display label
>
  {/* connection prose */}
</WisdomConnection>

// Reflection question (used in table context)
<ReflectionQuestion category={string}>
  {/* question text */}
</ReflectionQuestion>

// Callout block (used in Wisdom Briefs)
<WisdomCallout>
  {/* synthesis observation prose */}
</WisdomCallout>

// Key lesson (used in entry metadata display)
<KeyLesson number={number}>
  {/* lesson text */}
</KeyLesson>

// Practical application item
<PracticalApplication>
  {/* application prose */}
</PracticalApplication>

// Reading pathway step card (used in pathway guides)
<PathwayStep
  step={number}
  bookSlug={string}
  whyItBelongs={string}
  whatItBuilds={string}
/>
```

### 3.3 Theme Intelligence Page Content Model

Theme pages are partially database-driven (structured convergences/divergences JSON) and partially MDX (long-form prose sections). The `theme_pages.convergences_json` shape:

```json
{
  "convergences": [
    {
      "books": ["mans-search-for-meaning", "meditations"],
      "relationship_type": "HistoricalEcho",
      "label": "Frankl + Aurelius [Historical Echo]",
      "body": "The convergence between Aurelius and Frankl..."
    }
  ],
  "divergences": [
    {
      "label": "Interior vs. Relational",
      "body": "Frankl, Aurelius, Holiday vs. Arbinger, Greenleaf..."
    }
  ],
  "gateway_books": [
    {
      "book_slug": "mans-search-for-meaning",
      "rationale": "The philosophical foundation — provides the existential framework..."
    }
  ]
}
```

### 3.4 Reading Pathway Content Model

`pathway_books.why_it_belongs` and `what_it_builds` are plain text strings stored in the database. `pathway_lessons.accumulated_lesson` is plain text. All longer pathway prose (`intro_prose`, `closing_reflection`) is stored as markdown in the database, rendered via `next-mdx-remote` on the pathway page.

---

## Part IV — Navigation Structure

### 4.1 Primary Navigation

```
[ Leadership Through Transition ]    [ Library ]  [ Themes ]  [ Pathways ]  [ Wisdom Briefs ]  [ Discover ]
```

**Library** → `/library` — full book grid with filtering  
**Themes** → `/themes` — grid of 15 wisdom theme pages  
**Pathways** → `/pathways` — list of 10 reading pathways  
**Wisdom Briefs** → `/briefs` — synthesis documents  
**Discover** → `/discover` — AI-assisted reader matching

### 4.2 URL Structure

```
/                              — Homepage
/library                       — All books grid
/library/[slug]                — Book entry (e.g. /library/mans-search-for-meaning)
/library/[slug]#book-analysis  — Deep link to section
/themes                        — All themes grid
/themes/[slug]                 — Theme intelligence page (e.g. /themes/identity)
/pathways                      — All pathways list
/pathways/[slug]               — Pathway guide (e.g. /pathways/identity-beyond-rank)
/briefs                        — All wisdom briefs
/briefs/[slug]                 — Single brief (e.g. /briefs/identity-beyond-rank)
/discover                      — AI-assisted matching
/about                         — Platform purpose, constitution summary
/search?q=                     — Search results
```

### 4.3 Book Entry Page Navigation

The book entry page has two distinct navigation layers:

**In-page anchor navigation** (sticky sidebar or top bar on mobile):
- Opening Hook
- Book Analysis
- Leadership Transition Commentary
- Practical Applications
- Reflection Questions
- Cross-Book Wisdom
- Closing Reflection

**Cross-content navigation** (bottom of page):
- Related Books (5 cards)
- "Also appears in" (pathway and theme badges)
- Wisdom Engine connections (expandable)

### 4.4 Breadcrumbs

```
Library → Man's Search for Meaning
Themes → Identity → Man's Search for Meaning
Pathways → Identity Beyond Rank → Step 3: Man's Search for Meaning
```

### 4.5 Contextual Navigation Rules

The same book page renders differently based on the navigation path that led there. If a reader arrives at *Man's Search for Meaning* via the *Identity Beyond Rank* pathway, the page should:

- Show the pathway breadcrumb
- Display the pathway-specific "Why It Belongs" note
- Highlight the "What It Builds" for this step
- Show "Next in Pathway" navigation prominently

This context is passed as a URL parameter: `/library/mans-search-for-meaning?pathway=identity-beyond-rank&step=3`

---

## Part V — Search Design

### 5.1 Search Architecture

Two complementary search modes:

**Keyword search** — PostgreSQL full-text search over `books.search_vector`. The vector is built from a weighted concatenation:

```sql
-- Weight A (highest): title, author
-- Weight B: key lessons, section headers
-- Weight C: all section content
-- Weight D: keywords, categories, themes
UPDATE books SET search_vector =
  setweight(to_tsvector('english', coalesce(title,'')), 'A') ||
  setweight(to_tsvector('english', coalesce(author,'')), 'A') ||
  setweight(to_tsvector('english', coalesce(tagline,'')), 'B') ||
  setweight(to_tsvector('english', coalesce(array_to_string(
    ARRAY(SELECT keyword FROM book_keywords WHERE book_id = books.id), ' ')
  ,'')), 'C');
```

**Semantic search** — Vector embeddings stored in pgvector for concept-level search (e.g., "how to lead without authority" should surface *How to Lead When You're Not in Charge*, *The Servant as Leader*, and relevant sections of *Extreme Ownership*).

### 5.2 Search Scope

Search indexes the following content:
- Book titles, authors, taglines
- All 7 section bodies per complete entry
- Key lessons (all books)
- Theme page content
- Pathway guide content
- Wisdom Brief content
- Keywords and taxonomy tags

### 5.3 Search Result Types

Results are grouped by type in the UI:

```
BOOKS (12)
  Man's Search for Meaning — Viktor Frankl
  "...meaning cannot be pursued directly, only discovered..."

THEMES (2)
  Identity — Who are you when the rank is gone?
  Meaning — Why is the sacrifice worth enduring?

PATHWAYS (1)
  Identity Beyond Rank — For leaders navigating the loss...

WISDOM BRIEFS (1)
  What Great Books Teach About Identity Beyond Rank
```

### 5.4 Filters (Library Grid)

The `/library` grid supports the following filters. All are additive (AND logic within a group, OR across groups is available for multi-select):

| Filter Group | Options |
|---|---|
| Library Tier | Core · Supporting · Advanced · Specialized |
| Node Tier | Critical · High-Density · Standard · Pathway |
| Reading Pathways | All 10 approved pathways |
| Wisdom Themes | All 15 approved themes |
| Leadership Themes | All approved leadership themes |
| Transition Themes | All approved transition themes |
| Categories | All approved categories |
| Canon Status | Foundational · Military Leadership · Core Transition · etc. |
| Entry Status | Complete · Planned |
| Gold Standard | Yes / No |

**Default sort:** Node tier descending (Critical first), then alphabetical within tier.  
**Available sorts:** Alphabetical · Year (oldest first / newest first) · Recently Added · Most Connected (node_theme_count)

### 5.5 Search Autocomplete

Autocomplete suggests: book titles, author names, wisdom theme names, pathway names, and keywords. Maximum 8 suggestions, prioritized by node_tier then alphabetical.

---

## Part VI — Recommendation Engine

### 6.1 Philosophy

The recommendation engine is not an algorithmic black box — it is the Cross-Book Wisdom Engine expressed as navigation guidance. Every recommendation must be explainable in editorial terms: this book is recommended because it extends, converges with, or provides the philosophical foundation for what you just read.

The engine has three modes:

**Explicit connections** — drawn directly from `wisdom_connections` and `book_related` tables. These are editorially authored and the most trusted signal.

**Pathway sequencing** — if a reader is on a pathway, the next book in sequence is always the primary recommendation.

**Theme affinity** — books sharing ≥ 2 wisdom themes with the current book are secondary recommendations, ranked by node_theme_count.

### 6.2 Recommendation Surfaces

**On a Book Entry page — "What to Read Next" module:**

```
Priority 1: Next book in active pathway (if reader is following one)
Priority 2: The 5 Related Books from the entry (editorially authored)
Priority 3: Books connected via wisdom_connections from this entry (Convergence/Extension types first)
Priority 4: Theme-affinity books (≥ 2 shared wisdom themes, not already shown)

Maximum shown: 5 cards
```

**On a Theme page — "Start Reading" module:**

```
Show: The 4 Gateway Books for this theme (from theme_pages.gateway_books_json)
Then: Other books in this theme ordered by node_theme_count desc
```

**On a Pathway page — "Continue Reading" module:**

```
If reader has indicated progress: show next unread step
Otherwise: show Step 1
Sidebar: show intersecting pathways
```

**On the Homepage / Discover page:**

```
Reader self-selects from 4 entry point profiles
→ Routed to appropriate pathway or theme as described in §1.5
```

### 6.3 "Because You're Reading..." Logic

When a reader has a session (or uses the AI Discover tool), surface explicit "because you're reading X..." connections:

```
Reading Man's Search for Meaning?
→ "The Stoic tradition that Frankl draws from is most fully expressed in Meditations [Historical Echo · Character + Resilience]"
→ "Holiday's Ego Is the Enemy extends this into the specific vocabulary of transition [Extension · Identity + Resilience]"
```

Text is drawn directly from `wisdom_connections.connection_body`, attributed to the relationship type and themes.

### 6.4 Pathway Recommender Logic

Given a reader profile signal (from Discover assessment or session reading history), recommend a pathway using this decision tree:

```
If primary signal = identity disruption / "who am I without the rank"
  → Identity Beyond Rank

If primary signal = transition is imminent / retiring now
  → Preparing for Retirement

If primary signal = career / what do I do next
  → Building a Second Career

If primary signal = meaning / purpose / "what is it all for"
  → Finding Purpose After Service

If primary signal = leadership effectiveness / leading right now
  → Command Leadership or Leadership Foundations

If primary signal = uncertainty / chaos / don't know what's happening
  → Leading Through Uncertainty

If primary signal = mentorship / developing others
  → Developing Future Leaders
```

Multiple signals: offer the two most relevant pathways and let the reader choose.

---

## Part VII — Cross-Book Wisdom Engine Implementation

### 7.1 Architecture Overview

The Wisdom Engine is the platform's most distinctive feature. It is not a tag cloud or a "you might also like" module. It is a structured editorial knowledge graph, surfaced through the UI as three distinct experiences:

1. **Within-entry connections** — the 4 connections rendered in Section 6 of every book entry
2. **Theme convergence display** — the convergences/divergences on every Theme Intelligence Page
3. **Discovery graph** — the interactive cross-library exploration interface

### 7.2 Data Model (from §2.1)

The `wisdom_connections` table is the Wisdom Engine's core. Each row represents a directional connection from source book to target book, authored editorially per the Cross-Book Wisdom Framework standards:

- Named relationship type (one of the 7 approved types)
- Prose connection body (75–150 words, satisfying the three-part standard: specific idea + distinct contributions + synthesis observation)
- One or more wisdom theme tags
- Reference to which book entry it appears in

Connection count requirements per entry (from CrossBook Wisdom Framework §6.1):
- Foundational Canon: 4 connections minimum
- Core (non-Foundational): 4 connections minimum
- Supporting: 3 connections minimum
- Advanced: 3 connections minimum
- Specialized: 2 connections minimum

### 7.3 Rendering: Within-Entry Connections

Each `<WisdomConnection>` MDX component renders as an expandable panel:

```
┌─────────────────────────────────────────────────────────────────┐
│ Connection 1 — Meditations by Marcus Aurelius                   │
│ Historical Echo · Meaning + Character                    [+]    │
├─────────────────────────────────────────────────────────────────┤
│ [expanded]                                                      │
│ Aurelius and Frankl never encountered each other...             │
│                                                                 │
│ [→ Read Meditations entry]   [View in theme: Character]         │
└─────────────────────────────────────────────────────────────────┘
```

The relationship type badge uses a consistent color system:
- Convergence: Navy
- Extension: Slate
- HistoricalEcho: Gold
- TraditionBridge: Deep green
- DiagnosticPair: Purple
- Application: Mid-blue
- Contrast: Warm orange

### 7.4 Rendering: Theme Convergence Display

On Theme Intelligence Pages, convergences are rendered from `theme_pages.convergences_json`. Each convergence card shows:
- The books involved (with links)
- The relationship type badge
- The connection prose
- A "See this connection in context" link to the originating book entry

### 7.5 Rendering: Discovery Graph (Phase 2 Feature)

An interactive force-directed graph showing the full connection network. MVP requirements:

- Nodes = books, sized by node_theme_count
- Edges = wisdom_connections, colored by relationship_type
- Filterable by wisdom theme (shows only connections tagged with selected theme)
- Click node → navigate to book entry
- Hover edge → show connection label

Implementation: D3.js or Recharts force simulation, data from a `/api/wisdom-graph` endpoint that returns the full connection graph as `{ nodes: Book[], edges: WisdomConnection[] }`.

### 7.6 API Endpoints for Wisdom Engine

```
GET /api/wisdom-connections?bookSlug=mans-search-for-meaning
  → { connections: WisdomConnection[], connectedBooks: Book[] }

GET /api/wisdom-graph?theme=identity
  → { nodes: Book[], edges: WisdomConnection[] }

GET /api/book-cluster?bookSlug=meditations&depth=2
  → Books connected within 2 degrees via wisdom_connections
```

---

## Part VIII — Theme Page Architecture

### 8.1 Page Structure

Each of the 15 Theme Intelligence Pages follows this structure:

```
/themes/identity

[Theme Name: Large display heading]
[Governing Question: "Who are you when the rank is gone?"]
[One-paragraph theme definition]

━━━ Primary Books ━━━
[6-book grid — linked to entries]

━━━ Where the Authors Converge ━━━
[Convergence cards — from theme_pages.convergences_json]
[Each card: book names + relationship badge + prose + links]

━━━ Where the Authors Diverge ━━━
[Divergence rows — from theme_pages.divergences_json]
[Each: tension label + explanatory prose]

━━━ Leadership Application ━━━
[Prose from wisdom_themes.leadership_application]

━━━ Transition Application ━━━
[Prose from wisdom_themes.transition_application]

━━━ Gateway Books ━━━
[4-book grid with rationale — from theme_pages.gateway_books_json]

━━━ All Books in This Theme ━━━
[Full filterable list of books with this wisdom theme, sorted by node_tier]

━━━ Reading Pathways That Feature This Theme ━━━
[Pathway cards for pathways where this theme is prominent]

━━━ Wisdom Briefs ━━━
[Briefs that synthesize this theme, if any exist]

[Theme Evolution Notes — small text, editorial transparency]
```

### 8.2 Theme Index Page

`/themes` renders a grid of all 15 (eventually) theme pages. Each card shows:
- Theme name
- Governing question
- Primary book count
- 3 gateway book covers (placeholder imagery: geometric/typographic)

Ordered by the library's structural priority: Identity first (most connected), then Purpose, Meaning, Character, Resilience, Service, Trust, Accountability, Belonging, Leadership, Adaptability, Legacy, Mentorship, Reinvention, Communication.

### 8.3 Theme Status Tracking

Not all 15 themes have complete Theme Intelligence Pages yet. Status tracking:

| Status | Meaning | UI behavior |
|---|---|---|
| `active` | Full page built | Normal navigation |
| `partial` | Some sections complete | Show available sections, flag others as "coming soon" |
| `planned` | Theme defined, page not built | Show theme card with "Coming Soon" state |

Current active themes: Identity, Purpose, Meaning, Service, Trust, Accountability, Resilience, Leadership (8 of 15).

---

## Part IX — Reading Pathway Architecture

### 9.1 Page Structure

```
/pathways/identity-beyond-rank

[Pathway Name]
[Tagline: "For leaders navigating the loss of title..."]
[Reader Profile card — from pathways.reader_profile]
[Primary Need — from pathways.primary_need]
[Pathway Arc — visual timeline display]
[Estimated time: X–Y weeks]

━━━ Reading This Pathway ━━━
[Intro prose — from pathways.intro_prose]

━━━ The Reading Sequence ━━━
[Step cards, numbered 1–N]
  Each step:
  - Step number + Book title + Author + Year + Tier badge
  - Why It Belongs (from pathway_books.why_it_belongs)
  - What It Builds (from pathway_books.what_it_builds)
  - [→ Read this entry] button

━━━ Lessons Along the Journey ━━━
[Progressive reveal table — from pathway_lessons]
  "After Book 1 — [Name]": [accumulated_lesson]
  "After Books 1–2 — + [Name]": [accumulated_lesson]
  ...
  "Full Pathway — All N Books": [final lesson]

━━━ Closing Reflection ━━━
[Prose from pathways.closing_reflection]

━━━ Intersecting Pathways ━━━
[Cards for pathways in pathways.intersects_with]
```

### 9.2 Pathway Progress Tracking

Readers can optionally track their progress through a pathway without creating an account (using localStorage) or with an account (stored in database).

```sql
CREATE TABLE reader_pathway_progress (
  id                SERIAL PRIMARY KEY,
  reader_session_id VARCHAR(64),         -- anonymous session
  reader_id         INTEGER,             -- authenticated user (optional)
  pathway_id        INTEGER REFERENCES pathways(id),
  completed_steps   INTEGER[],           -- array of step_numbers completed
  started_at        TIMESTAMPTZ DEFAULT NOW(),
  updated_at        TIMESTAMPTZ DEFAULT NOW()
);
```

Progress state drives:
- "Continue Reading" next-step recommendations
- Visual step completion indicators on pathway page
- "Books read in this pathway" count on library grid

### 9.3 Pathway Entry from Book Page

When a book page is accessed with `?pathway=[slug]&step=[n]` context (set when a reader clicks a pathway step):

```tsx
// app/library/[slug]/page.tsx
export default function BookEntryPage({ params, searchParams }) {
  const pathwayContext = searchParams.pathway && searchParams.step
    ? { slug: searchParams.pathway, step: parseInt(searchParams.step) }
    : null;

  // If pathwayContext exists:
  // - Show pathway breadcrumb
  // - Show "Why It Belongs" panel
  // - Show "What It Builds" panel
  // - Show "Next in Pathway" CTA at bottom
}
```

### 9.4 Pathway Index Page

`/pathways` lists all 10 pathways. Each card:
- Pathway name
- Tagline
- Book count + estimated weeks
- 3 book covers from the sequence
- "Who This Is For" one-line summary

Grouped by primary purpose: Identity/Purpose pathways first, then Effectiveness pathways, then Specialized pathways.

---

## Part X — AI Assistant Architecture

### 10.1 Philosophy

The LTT AI assistant ("the Library's voice") is not a general-purpose chatbot. It has one job: help readers navigate the library in a way that the editorial voice would endorse. It speaks in the LTT voice — reflective, grounded, honest, not inspirational-speech mode.

It knows three things deeply: the book entries, the wisdom connections, and the pathway sequences. It does not summarize books on demand (that would undermine the platform's purpose), but it does help readers understand where to start, what connects to what, and which books speak to their specific situation.

### 10.2 Capabilities

| Capability | Implementation |
|---|---|
| Reader matching | Takes reader's self-described situation → recommends 1–2 pathways or entry books |
| Connection explanation | "How are Frankl and Aurelius connected?" → draws from `wisdom_connections` + theme data |
| Pathway navigation | "I'm on Identity Beyond Rank. What should I read next?" → checks progress, returns next step with context |
| Book comparison | "What's the difference between Start With Why and The Infinite Game?" → draws from both entries + connection data |
| Theme exploration | "Tell me more about the Belonging theme" → draws from `wisdom_themes` + theme pages |
| Reading time | "How long will it take to finish the Command Leadership pathway?" → from `pathways.reading_weeks_min/max` |

**Not capable of (by design):**
- Providing plot summaries or chapter-by-chapter content (not a summary service)
- Giving personal life advice beyond book recommendations
- Rating or ranking books comparatively ("which is better")
- Discussing books not in the LTT library

### 10.3 System Prompt

The AI assistant is backed by a retrieval-augmented generation (RAG) architecture. The system prompt defines the voice and constraints:

```
You are the editorial voice of Leadership Through Transition, a curated library platform for military leaders navigating separation and transition to civilian life.

Your role is to help readers navigate the library — not to summarize books, not to give life advice, not to inspire. You help readers find the right book, understand why books connect to each other, and navigate the reading pathways.

Voice: Reflective. Grounded. Honest. You speak the way a senior leader with thirty years of experience and wide reading would speak — thoughtful, specific, without grand declarations or motivational language.

You have access to:
- Complete metadata and editorial notes for all library books
- All cross-book wisdom connections with their relationship types and themes
- All reading pathway sequences and their rationale
- All theme intelligence page content

When recommending books, always explain WHY — not what the book is about, but why it belongs in the reader's current situation. Draw this explanation from the editorial content, not from generic summaries.

When explaining connections between books, use the Cross-Book Wisdom Framework language: relationship type, specific idea shared, what each book contributes distinctly, and what the convergence reveals.

Do not recommend books outside the LTT library. If asked about a book not in the library, acknowledge it may be valuable, note that it is not currently in the library, and offer the closest LTT equivalent.

Maximum response length: 300 words. If a question requires more, offer to continue.
```

### 10.4 RAG Architecture

```
Reader query
    ↓
Query analysis (intent classification)
    ↓
Retrieval layer:
  - Keyword search over book entries, theme pages, pathway content
  - Semantic search (embeddings) for concept-level matching
  - Structured lookup for specific book/theme/pathway queries
    ↓
Context assembly (top-k relevant chunks + structured metadata)
    ↓
LLM completion with system prompt + retrieved context
    ↓
Response
```

**Vector store:** pgvector extension on the same PostgreSQL database. Embed:
- Each book section (7 sections × 40 books = 280 chunks)
- Each theme page section
- Each pathway step's `why_it_belongs` + `what_it_builds`
- Each wisdom connection `connection_body`

**Embedding model:** `text-embedding-3-small` (OpenAI) or equivalent. Chunk size: ~500 tokens with 50-token overlap.

### 10.5 Discover Flow (Structured Assessment)

The `/discover` page offers a structured 3-question reader assessment before opening the full chat:

```
Question 1: "Where are you right now?"
  Options:
  [ ] Retirement papers are in — transition is happening now
  [ ] I'm thinking about it, timeline unclear
  [ ] I've separated and I'm navigating civilian life
  [ ] I'm still active duty and want to prepare
  [ ] I've been out for a while and need a reset

Question 2: "What feels most urgent?"
  Options:
  [ ] I'm not sure who I am without the rank
  [ ] I need to figure out what I'm for now
  [ ] I need to be effective in a civilian job fast
  [ ] I want to lead better, rank or no rank
  [ ] I'm not sure — that's why I'm here

Question 3: "How do you read?"
  Options:
  [ ] One book at a time, cover to cover
  [ ] I want a guided sequence — tell me exactly what order
  [ ] I want to explore by topic
  [ ] I want to start with whatever is most important
```

Answers map to pathway and/or book recommendations per the logic in §6.4.

### 10.6 Assistant UI

The assistant lives in three contexts:
- **Standalone at `/discover`** — full-width, assessment-first
- **Library sidebar** — narrow panel at right of book entry page, context-aware (knows which book is being read)
- **Pathway page sidebar** — context-aware (knows which pathway and step)

State management: The assistant maintains conversation context within a session. The currently-viewed book/pathway/theme is always injected into the context as a system message: `"The reader is currently viewing: [Book Title]. Answers should be relevant to this context where possible."`

---

## Part XI — Next.js Implementation Notes

### 11.1 Project Structure

```
src/
├── app/
│   ├── page.tsx                     # Homepage
│   ├── library/
│   │   ├── page.tsx                 # Library grid
│   │   └── [slug]/
│   │       └── page.tsx             # Book entry
│   ├── themes/
│   │   ├── page.tsx                 # Themes index
│   │   └── [slug]/
│   │       └── page.tsx             # Theme page
│   ├── pathways/
│   │   ├── page.tsx                 # Pathways index
│   │   └── [slug]/
│   │       └── page.tsx             # Pathway guide
│   ├── briefs/
│   │   ├── page.tsx                 # Briefs index
│   │   └── [slug]/
│   │       └── page.tsx             # Single brief
│   ├── discover/
│   │   └── page.tsx                 # AI discovery
│   ├── search/
│   │   └── page.tsx                 # Search results
│   └── api/
│       ├── books/route.ts
│       ├── wisdom-connections/route.ts
│       ├── wisdom-graph/route.ts
│       ├── search/route.ts
│       └── assistant/route.ts        # Streaming AI responses
│
├── components/
│   ├── ltt/                         # LTT MDX components (§3.2)
│   │   ├── WisdomConnection.tsx
│   │   ├── ReflectionQuestion.tsx
│   │   ├── WisdomCallout.tsx
│   │   ├── KeyLesson.tsx
│   │   ├── PracticalApplication.tsx
│   │   └── PathwayStep.tsx
│   ├── library/
│   │   ├── BookGrid.tsx
│   │   ├── BookCard.tsx
│   │   ├── BookFilters.tsx
│   │   └── SectionNav.tsx
│   ├── wisdom/
│   │   ├── WisdomEnginePanel.tsx
│   │   ├── ConvergenceCard.tsx
│   │   └── WisdomGraph.tsx          # Phase 2
│   ├── discovery/
│   │   ├── ReaderAssessment.tsx
│   │   └── AssistantChat.tsx
│   └── navigation/
│       ├── SiteNav.tsx
│       └── Breadcrumbs.tsx
│
├── lib/
│   ├── db/
│   │   ├── books.ts                 # Book queries
│   │   ├── themes.ts                # Theme queries
│   │   ├── pathways.ts              # Pathway queries
│   │   ├── wisdom.ts                # Wisdom engine queries
│   │   └── search.ts                # Search queries
│   ├── mdx/
│   │   ├── render.ts                # MDX rendering pipeline
│   │   └── components.ts            # MDX component registry
│   ├── ai/
│   │   ├── retrieval.ts             # RAG retrieval logic
│   │   ├── embeddings.ts            # Embedding generation
│   │   └── assistant.ts             # Assistant chat logic
│   └── catalog/
│       └── seed.ts                  # Catalog seed from JSON
│
├── content/                         # MDX source files (alternative to DB storage)
│   ├── books/
│   │   └── mans-search-for-meaning/ # Optional: MDX files per section
│   └── briefs/
│
└── types/
    ├── book.ts
    ├── theme.ts
    ├── pathway.ts
    └── wisdom.ts
```

### 11.2 Static Generation Strategy

| Route | Generation Strategy | Revalidation |
|---|---|---|
| `/library` | `generateStaticParams` + ISR | 1 hour |
| `/library/[slug]` | `generateStaticParams` for complete entries | On content update |
| `/themes/[slug]` | Static for active themes | On content update |
| `/pathways/[slug]` | Static for all pathways | On content update |
| `/briefs/[slug]` | Static | On content update |
| `/discover` | Client-side | N/A |
| `/search` | Client-side | N/A |

Book entries are pre-rendered at build time for all `entry_status = 'complete'` books. The `/library` grid uses ISR to reflect newly published entries without full rebuilds.

### 11.3 MDX Rendering Pipeline

Book section content is stored as MDX strings in `book_sections.content_mdx`. Rendering:

```tsx
// lib/mdx/render.ts
import { compileMDX } from 'next-mdx-remote/rsc';
import { lttComponents } from './components';

export async function renderBookSection(mdxContent: string) {
  const { content } = await compileMDX({
    source: mdxContent,
    components: lttComponents,
    options: { parseFrontmatter: false }
  });
  return content;
}
```

Each of the 7 sections is rendered independently and composed on the page. This allows sections to be independently cached, selectively revalidated, and rendered in different UI contexts (e.g., the `leadership_transition_commentary` section can be surfaced in a pathway context with additional annotations).

### 11.4 Book Entry Page — Section Composition

```tsx
// app/library/[slug]/page.tsx
export default async function BookEntryPage({ params, searchParams }) {
  const book = await getBookWithSections(params.slug);
  const pathwayCtx = getPathwayContext(searchParams);
  const wisdomConnections = await getWisdomConnections(book.id);

  return (
    <article>
      <BookHeader book={book} pathwayCtx={pathwayCtx} />
      <TaxonomyTags book={book} />
      <KeyLessons lessons={book.key_lessons} />
      <SectionNav sections={SECTION_KEYS} />

      <section id="opening-hook">
        {await renderBookSection(book.sections.opening_hook)}
      </section>

      <section id="book-analysis">
        {await renderBookSection(book.sections.book_analysis)}
      </section>

      <section id="leadership-transition-commentary">
        {await renderBookSection(book.sections.leadership_transition_commentary)}
      </section>

      {/* ... remaining sections ... */}

      <section id="cross-book-wisdom">
        <WisdomEnginePanel
          connections={wisdomConnections}
          currentBook={book}
        />
      </section>

      <RelatedBooks books={book.related_books} />
      <PathwayBadges pathways={book.pathways} />
      {pathwayCtx && <PathwayNextStep ctx={pathwayCtx} book={book} />}
    </article>
  );
}
```

### 11.5 Environment Variables

```bash
DATABASE_URL=postgresql://...
OPENAI_API_KEY=...
NEXT_PUBLIC_SITE_URL=https://leaderthroughtransition.com
```

### 11.6 Metadata and SEO

Each book entry page generates structured metadata:

```tsx
export async function generateMetadata({ params }) {
  const book = await getBook(params.slug);
  return {
    title: `${book.title} — Leadership Through Transition`,
    description: book.tagline,
    openGraph: {
      title: book.title,
      description: book.tagline,
      type: 'article',
    },
    // JSON-LD structured data
    other: {
      'application/ld+json': JSON.stringify({
        "@context": "https://schema.org",
        "@type": "Book",
        "name": book.title,
        "author": { "@type": "Person", "name": book.author },
        "datePublished": book.year,
      })
    }
  };
}
```

---

## Part XII — Document Coherence Revisions

The following revisions are recommended to align source documents with this specification before development begins. These are editorial corrections that bring earlier documents into coherence with the Book Entry data (the most authoritative source).

### 12.1 CrossBook Wisdom Framework — Node Map Updates

**Current → Corrected:**

| Book | Current Classification | Corrected Classification | Reason |
|---|---|---|---|
| *Leadership and Self-Deception* | Standard Node (listed, 4 themes) | High-Density Node (4 themes) | 4 wisdom themes = High-Density per spec definition |
| *The Infinite Game* | Standard Node (listed, 4 themes per entry) | High-Density Node (4 themes) | Book Entry records 4 wisdom themes; framework note says "Standard (3)" which was an error |

**Action:** Update Table in CrossBook Wisdom Framework §5.2 Node Map for both entries.

### 12.2 Library Architecture Document — Tier Corrections

**Current → Corrected:**

| Book | Library Architecture Tier | Corrected Tier | Authority |
|---|---|---|---|
| *Start With Why* | Supporting | Core / Foundational Canon | Book Entry v1.0 |
| *The Infinite Game* | Supporting | Supporting + Core Transition Canon | Book Entry v1.0 |

**Action:** Update Phase 3 book listings in Library Architecture. Note that the Library Architecture document predates the Book Entries; the Book Entries are the canonical record going forward.

### 12.3 MasterCatalog_v1 Supersession

The `LTT_MasterCatalog_v1.json` and `LTT_MasterCatalog_v1.xlsx` generated in June 2025 reflect the corrected Book Entry data and should be treated as the canonical catalog reference. They supersede any implicit taxonomy in the Library Architecture document where the two conflict.

### 12.4 Editorial Authority Order (Formal Statement)

For any future conflicts between project documents, this authority order governs:

1. **Individual Book Entries** (most specific, most recent, Gold Standard reviewed)
2. **Cross-Book Wisdom Framework** (structural rules for connections)
3. **Book Entry Blueprint** (format and quality standards)
4. **Voice & Style Guide** (tone and language)
5. **Taxonomy & Content Standards** (approved vocabulary)
6. **Project Constitution** (mission and principles)
7. **Library Architecture Document** (strategic framing, superseded by Book Entries on metadata)
8. **Master Catalog** (derived record, reflects Book Entry data)

---

## Part XIII — Implementation Phases

### Phase 1 — Foundation (Weeks 1–6)

**Goal:** A fully functional library of 15 complete entries, browseable and searchable.

- Database setup and seed (all 40 book records, 15 complete entry sections)
- Book entry pages (all 15 complete entries rendered via MDX)
- Library grid with filtering
- Basic keyword search
- 6 reading pathway pages
- 8 active theme pages
- Navigation and breadcrumbs
- SEO metadata

**Not in Phase 1:** AI assistant, wisdom graph visualization, reader progress tracking, reader accounts

### Phase 2 — Wisdom Engine (Weeks 7–10)

**Goal:** The cross-book wisdom network is navigable and discoverable.

- Wisdom connections rendered in all complete entries
- Theme convergence/divergence display on theme pages
- "Because You're Reading..." contextual recommendations
- Theme-based filtering on library grid
- Wisdom Brief published (Identity Beyond Rank)

### Phase 3 — Discovery (Weeks 11–16)

**Goal:** The AI assistant and personalized discovery experience.

- `/discover` page with reader assessment
- AI assistant (RAG architecture, pgvector embeddings)
- Pathway progress tracking (localStorage)
- "Next in Pathway" contextual navigation
- Semantic search

### Phase 4 — Growth (Ongoing)

**Goal:** Scale to 40 complete entries and full feature maturity.

- Remaining 25 book entries (as written)
- All 15 theme pages active
- 10 reading pathways complete
- Reader accounts and progress persistence
- Wisdom graph visualization
- Additional Wisdom Briefs

---

## Appendix A — Approved Vocabulary Reference

The following lists are authoritative and should be used verbatim as enum values in the database.

**Library Tiers:** Core · Supporting · Advanced · Specialized

**Node Tiers:** Critical Node · High-Density Node · Standard Node · Pathway Node

**Canon Designations:** Foundational Canon · Philosophy & Meaning Canon · Military Leadership Canon · Core Leadership Canon · Core Transition Canon · Sinek Trilogy Completion

**Wisdom Themes (15):** Identity · Purpose · Meaning · Character · Resilience · Accountability · Service · Adaptability · Trust · Mentorship · Legacy · Belonging · Reinvention · Leadership · Communication

**Relationship Types (7):** Convergence · Extension · Contrast · DiagnosticPair · Application · HistoricalEcho · TraditionBridge

**Reading Pathways (10):** Preparing for Retirement · Identity Beyond Rank · Finding Purpose After Service · Building a Second Career · Leading Through Uncertainty · Developing Future Leaders · Resilience Under Pressure · Leadership Foundations · Command Leadership · Personal Growth & Reflection

**Entry Status:** complete · draft · planned

**Gold Standard:** Boolean (TRUE = Gold Standard designation confirmed by formal review)

---

## Appendix B — Content Completeness Status at Spec Publication

| Content Type | Complete | Planned | Notes |
|---|---|---|---|
| Book Entries | 15 | 25 | See Library Architecture for planned titles |
| Theme Pages | 8 | 7 | Active: Identity, Purpose, Meaning, Service, Trust, Accountability, Resilience, Leadership |
| Pathway Guides | 6 | 4 | Complete: Preparing for Retirement, Identity Beyond Rank, Finding Purpose After Service, Leading Through Uncertainty, Developing Future Leaders, Command Leadership. Planned: Building a Second Career, Resilience Under Pressure, Leadership Foundations, Personal Growth & Reflection |
| Wisdom Briefs | 1 | ~9 | Identity Beyond Rank complete |
| Master Catalog | v1.0 | — | 15 entries; update as entries complete |

---

*Leadership Through Transition · Website Build Package · Version 1.0 · June 2025*
