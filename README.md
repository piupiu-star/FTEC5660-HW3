# FTEC5660 Homework 3: Fisher's 15-Point Investment Agent

Build a LangChain agent that researches a company with tools and applies the
framework from Philip Fisher's *Common Stocks and Uncommon Profits* (Part One,
Chapter 3 "What to buy: the fifteen points to look for in a common stock", and
Chapter 4 "What to buy: applying this to your own needs"). You choose 3 of
Fisher's 15 points and decide yourself what PASS means for each. For every
company your agent gives a PASS/FAIL verdict on your 3 points and a short
summary.

The companies are fictional, so the model cannot answer from memory. Unlike a
pile of reports, the evidence is spread over three sources, and your agent has
to go and get it with tools, the way Fisher's "scuttlebutt" method sends an
investor to talk to customers, competitors, suppliers and staff.

## Data and tools

`public_test/` has one folder per company. Your code receives four tools per
company (built by the provided code in `hw3.py`):

| Tool | What it gives you |
| --- | --- |
| `list_documents()` | the company's six documents and their dates |
| `read_document(name)` | one document: the 2025 letter to shareholders, four earnings calls (Q3 2025 to Q2 2026) and a news file (July 2025 to July 2026) |
| `sql_query(query)` | one read-only SQLite query on the company's database; at most 50 rows come back, so use `SUM`, `GROUP BY` and `WHERE` |
| `interview(role, topic)` | what one person says about the company; at most **10 interviews per company** |

The documents are what management says. Management's own claims (such as "we
have never cut research" or "our people are our greatest strength") are not
evidence on their own: check them against the database and the interviews.

**Database tables** (amounts in USD thousands; dates `YYYY-MM-DD`, months
`YYYY-MM`, quarters like `2025Q3`; the fiscal year is the calendar year):

| Table | Columns |
| --- | --- |
| `company` | name, industry |
| `industry_medians` | industry, revenue_growth_2y_pct, new_product_share_pct, new_product_sales_per_rd, operating_margin_pct (one row per industry; use the row for the company's own industry) |
| `orders` | order_id, order_date, product, customer, amount, note (about 3,000 to 4,000 orders from January 2023 to June 2026; revenue is the sum of order amounts) |
| `products` | product, launch_date |
| `monthly_costs` | month, cost_of_sales, selling_admin, research_development |
| `quarterly_cash` | quarter, operating_cash_flow, capex, cash_end |
| `capex_plan` | year, planned_capex (2026 to 2028) |
| `returns` | month, units_shipped, units_returned |
| `insider_trades` | trade_date, person, title, side (`buy` or `sell`), shares, price |

**Terms used in the data.**

| Term | Meaning |
| --- | --- |
| FY2025 | Fiscal year 2025. Here the fiscal year is the calendar year, so FY2025 is January to December 2025. |
| Quarter | Three months of the year: Q1 is January to March, Q2 April to June, Q3 July to September and Q4 October to December. The database writes `2025Q3`; the documents write "third quarter 2025". |
| Amounts | The database is in **USD thousands** (676,100 means US$676.1 million). The documents use USD millions. |
| Revenue | Money from sales: the sum of `amount` in `orders`. |
| Cost of sales | The cost of making the products that were sold. |
| Selling and admin | The cost of selling and of running the company (sales staff, offices, management). |
| R&D | Research and development spending, mainly on new products. |
| Operating margin | Operating profit as a share of revenue: (revenue − cost of sales − selling and admin − R&D) ÷ revenue. |
| Operating cash flow | Cash that the day-to-day business brings in during a quarter. |
| Cash at the end of a quarter | The cash the company holds on the last day of that quarter (`cash_end`). |
| Capex | Capital expenditure: spending on factories, machines and other long-term assets. `capex_plan` holds the planned capex for 2026 to 2028. |
| Return rate | Units returned ÷ units shipped, from the `returns` table. |
| Industry median | The middle value for companies in the same industry, from `industry_medians`. |
| Insider | A director or senior manager of the company. `insider_trades` lists the shares they bought (`buy`) or sold (`sell`). |
| Earnings call | A meeting each quarter where management presents the quarter's results to investors. Each call quotes that quarter's revenue. |

**Interviews.** Roles: `customer`, `current_employee`, `former_employee`,
`competitor`, `supplier`. Topics: `sales_team`, `working_conditions`,
`top_management`, `decision_making`, `competitive_edge`. Only some people know
about each topic; the others say they can't comment, and that answer still
uses up one interview.

| Topic | Who knows about it |
| --- | --- |
| `sales_team` | customer, competitor |
| `working_conditions` | current_employee, former_employee |
| `top_management` | former_employee, supplier |
| `decision_making` | supplier, customer |
| `competitive_edge` | competitor, customer |

People who know about a topic always agree with each other, so one good answer
is enough.

## Choose your 3 points and your rules

Pick any 3 of Fisher's 15 points and write them in `CHOSEN_POINTS` near the
top of `hw3.py`, e.g. `CHOSEN_POINTS = [1, 7, 15]`. The full questions are in
`FISHER_POINTS` in `hw3.py` and in the book.

| # | Fisher's point | Where the evidence is |
| --- | --- | --- |
| 1 | Market potential for sales growth | database |
| 2 | New products as current lines mature | database |
| 3 | Effective research and development | database |
| 4 | Above-average sales organization | interviews (`sales_team`) |
| 5 | Worthwhile profit margin | database |
| 6 | Maintaining or improving margins | database |
| 7 | Labour and personnel relations | interviews (`working_conditions`) |
| 8 | Executive relations | interviews (`top_management`) |
| 9 | Depth of management | interviews (`decision_making`) |
| 10 | Cost analysis and accounting controls | earnings calls, checked against the database |
| 11 | Advantage over competitors | interviews (`competitive_edge`) |
| 12 | Long-range outlook on profits | database |
| 13 | Growth without dilutive equity financing | database |
| 14 | Management talks freely about troubles | earnings calls, checked against the database |
| 15 | Management of unquestionable integrity | news file, checked against the database |

For each of your points, **you design the rule**: which evidence to use, what
to compute or whom to ask, and where the line between PASS and FAIL lies. PASS
means the company looks favourable on that point, as Fisher would see it.

We grade your verdicts against our own answer key. It uses one fixed rule per
point, which we do not publish, and the data never puts a company close to
the line of that rule, so a sensible reading of Fisher usually gives the same
answer. **Test your rules on the public answers**:
`public_test/ground_truth.json` gives the expected verdict on all 15 points
for each of the 20 public companies, and every point fails in 6 of them. If
your rule disagrees with the public answers, rethink it.

**Tip: calculate with tools, not in your head.** Do every sum, ratio and
comparison in SQL or Python, including the number of days between two dates
(SQLite's `julianday()` gives a date as a number of days).

## Student task

Fill in the three places in `hw3.py` marked `### YOUR CODE HERE`. You may add
imports, constants and helper functions above them, but do not change the
provided code below them:

- `CHOSEN_POINTS` lists your 3 points.
- `build_agent()` creates your model and anything else you need once.
- `analyze_companies(agent, companies)` receives
  `{company_folder: [list_documents, read_document, sql_query, interview]}`
  and returns, for every company,
  `{"points": {1: "PASS", 7: "FAIL", 15: "PASS"}, "summary": "..."}` (here with
  `CHOSEN_POINTS = [1, 7, 15]`).

Use the course techniques that fit your points: tool use for the database and
the interviews, planning which interviews to spend, routing each point to the
right source, prompt chaining, parallel calls, reflection to check your
answers, or a combination. Your code must read company data **only through
the four tools**; do not open `company.db` or `interviews.dat` yourself. Do
not hard-code folder names, company names or the public answers.

## Setup and public test

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
echo "DEEPSEEK_API_KEY=your_key_here" > .env
python3 hw3.py --data-folder public_test
```

Use your own DeepSeek API key (from https://platform.deepseek.com). The
required model is `deepseek-v4-flash`. We grade with our own key, so never put
a key in your code; `.env` must stay out of git.

The program creates `results.csv` with the columns `company`, `points`,
`points_correct`, `point_errors`, `interviews_used` and `summary`, and prints
the point score on your 3 points (the percentage after "balanced"). The point
score is the average of two shares: the share of truly passing verdicts that
you marked PASS and the share of truly failing verdicts that you marked FAIL.
For example, 90% and 60% give (90% + 60%) / 2 = 75%, and marking every point
PASS gives (100% + 0%) / 2 = 50%. Once `CHOSEN_POINTS` is set, the starter
runs before you add any API code (it returns no verdicts, so it scores 0%).

`public_test/ground_truth.json` gives the expected verdicts on all 15 points
for each public company. Grading uses a private set of about 20 unseen
companies with the same tools and the same answer key rules, but different
names and wording, and scores your 3 points only.

## Homework 3 solution:

### My 3 points and rules

I chose points 1, 5 and 11 because between them they force the agent to work
from two kinds of evidence at once (two database points, one interview point),
and because each of them has a line that can be drawn before any company is
seen. The two database points are cheap and exact; the interview point answers
something the accounts cannot.

**Point 1 — "Does the company have products or services with sufficient market
potential to make possible a sizable increase in sales for at least several
years?"**

*Why.* It is the gate every other point sits behind: potential that never turns
into sales is not potential. It is also the point where management's own words
are least trustworthy, because every letter to shareholders claims growth.

*PASS rule.* PASS when revenue from FY2023 to FY2025 grew by more than the
median firm in the company's own industry, i.e. when
`(revenue_2025 / revenue_2023 - 1)` exceeds `industry_medians.revenue_growth_2y_pct`
for that industry.

*Evidence.* One SQL aggregate over `orders` for the two fiscal years, plus the
`industry_medians` row. Note that the data ends in June 2026, so 2026 is never
compared with a full year: the rule uses FY2023 and FY2025 only.

*Why relative rather than absolute.* Fisher asks about potential "for at least
several years", and the industry median is the only honest measure of whether a
firm is capturing the potential that is demonstrably there. A firm that grows
14% in an industry growing 22% is losing ground even though the number looks
healthy.

**Point 5 — "Does the company have a worthwhile profit margin?"**

*Why.* It is the cleanest test of whether the business is worth owning, and it
is the natural partner of point 1: growth bought with margin is not progress.

*PASS rule.* PASS when the FY2025 operating margin,
`(revenue - cost_of_sales - selling_admin - research_development) / revenue`,
is above the industry median `operating_margin_pct`.

*Evidence.* One SQL statement: revenue from `orders`, the three cost lines from
`monthly_costs`, the median from `industry_medians`. The arithmetic happens in
Python, never in the model.

*Why not a fixed bar.* A fixed threshold (say 10%) gets the public companies
wrong in both directions: Northwind Group earns 12.8% but its industry median is
15.1%, while Everline Dynamics earns 6.3% against a 10.3% median. "Worthwhile"
only means something relative to what the business is capable of earning.

**Point 11 — "Are there other aspects of the business, somewhat peculiar to the
industry involved, which will give the investor important clues as to how
outstanding the company may be in relation to its competition?"**

*Why.* It is the one chosen point that accounting cannot answer: an order book
looks identical whether the product is unique or a commodity. Only the people on
the other side of the table know, which is exactly what Fisher's scuttlebutt
method is for.

*PASS rule.* PASS when a competitor or a customer describes a durable advantage:
switching is slow or costly, know-how or the installed base is hard to copy,
rivals cannot match the performance, or customers who leave come back. FAIL when
the same people describe a commodity: interchangeable products, buyers choosing
on price, a rival who says nothing is hard to copy.

*Evidence.* `interview("competitor", "competitive_edge")`, falling back to
`interview("customer", "competitive_edge")` when the first person says they
cannot comment. Two of the ten interviews are the most this point can ever cost.

### Design

```
                     four LangChain tools, one set per company
   ┌───────────────┬───────────────┬────────────┬──────────────────┐
   │ list_documents│ read_document │ sql_query  │    interview     │
   └───────────────┴───────────────┴─────┬──────┴────────┬─────────┘
        (not needed for these 3 points)  │               │
                    ┌────────────────────┴───┐    ┌──────┴────────────────┐
                    │ SQL_COMPANY            │    │ competitor            │
                    │ SQL_REVENUE_GROWTH     │    │   -> customer         │
                    │ SQL_OPERATING_MARGIN   │    │ (skip "cannot comment")│
                    └────────────┬───────────┘    └──────┬────────────────┘
                                 │                       │
                    Python arithmetic + threshold        │  DeepSeek judge (JSON)
                    compare with industry median         │        │
                                 │                       │  DeepSeek reflection
                                 │                       │  (independent second look)
                    ┌────────────┴───────────────────────┴───┐
                    │ evidence list with the verdicts fixed   │
                    └────────────────────┬───────────────────┘
                                         │  DeepSeek writes 2-3 sentences
                                    results.csv (summary)
```

The pipeline is a router, not a chat. Every company is handled by
`_analyze_one_company`: three SQL statements go through the `sql_query` tool and
their numbers are compared with the industry median **in Python** — the model
never sees a number it could recompute, which removes the single largest source
of wrong verdicts. Point 11 takes the other branch: one targeted interview with
the person listed as knowing the topic, one model call that must answer with
JSON (`PASS`/`FAIL` plus a short reason), then a second model call that re-reads
the quote without being told what to conclude and may replace the first verdict.
The fixed verdicts and the evidence that produced them are then handed to a
third model call whose only job is prose, under explicit instructions not to
recompute or change anything. Companies run concurrently (semaphore of 4), each
company is wrapped in `try/except`, and the model calls are wrapped again so
a failed API call costs one point at most: the two SQL points and a summary
built from the evidence are still written. `CHOSEN_POINTS`, the SQL and the
thresholds are the only place the rules live; folder names, company names and
public answers never appear in the code.

### Results

Point score on `public_test` (3 consecutive runs):

| run | point accuracy | balanced score | wall clock |
| --- | --- | --- | --- |
| 1 | 60 / 60 | **100.0%** | 23.7 s |
| 2 | 60 / 60 | **100.0%** | 23.0 s |
| 3 | 60 / 60 | **100.0%** | 23.5 s |

Each run answers all 3 points on all 20 companies, uses exactly one interview
per company (out of a budget of ten) and finishes in about 23 seconds, far inside
the 30 minute limit.

**Companies where my verdicts disagree with the public answers: none.** All
three points agree with `ground_truth.json` on all 20 companies. That is the
expected outcome rather than a lucky one: every point fails in exactly 6 of the
20 companies in the answer key, and in the public data no company sits near the
line of a relative rule. The closest calls are still comfortable — Kestrel
Technologies grows 4.2% against a 9.8% median, Everline Dynamics earns 6.3%
against 10.3% — so a small change in wording in the private set should not flip
a verdict.

**Negative control.** With an invalid API key (so every model call fails) the
same code still finishes, still writes `results.csv`, and scores 54/60
(balanced 83.3%): the two database points are fully correct and point 11 falls
back to `PASS`. That is the behaviour I wanted — the expensive, uncertain part
of the agent can fail without destroying the cheap, certain part.

### What I would do differently

A first version of point 3 (R&D effectiveness) compared the new-product share
of revenue with the industry median and disagreed with 13 of the 20 public
answers. The model could have written a persuasive paragraph for every one of
those wrong verdicts: the failure was in the rule, not in the reasoning, and
only the public answers exposed it. That is why the final rules are all
relative to the industry row, and why every comparison is executed, not
narrated.

## Task 2: Reflection on AI sycophancy and AI psychosis

*Sycophancy* is the model's tendency to agree with whatever framing it is
given. *AI psychosis* is the more serious failure: the model keeps producing
confident, self-consistent output that has lost contact with the world, and each
step reinforces the last.

**Sycophancy showed up in my own design, not in a hypothetical.** The
reflection step is where it would have hurt: the cheap way to ask for a second
opinion is "my colleague says FAIL, do you agree?", and a sycophantic model
agrees about as often as it is asked. I deliberately did not do that. The
verification prompt re-states the criterion — durable advantage versus
commodity — and asks the model to read the quote on its own, and only afterwards
compares with the first verdict. Even so, the second call can see the first
verdict in its input, and that anchoring is a known cost of the design. A
sceptical version would run the two judgements in parallel with no shared
context and break ties with a third call. The same pressure exists one level up:
once I had written "PASS means growth above the industry median", the model had
every incentive to find growth above the median in the SQL output. I removed
that possibility structurally by never letting the model see the numbers, and by
making the verdict a two-line `if` in Python. Sycophancy needs a sentence to
agree with; the deterministic layer does not speak.

**AI psychosis is the failure I saw happen.** During development I ran the
pipeline before the API key was valid. Every model call returned a 401, every
company got the fallback `PASS` on the interview point, and the run still
printed a tidy `results.csv` with a score of 83.3%. Nothing in the output looked
wrong. A reader of that file would have concluded that thirteen companies have a
durable advantage over their competitors, when in fact nothing had been
observed at all. That is the mechanism: absence of evidence does not produce
silence, it produces a plausible default, and the default becomes data for
everything downstream. The same mechanism made the wrong point-3 rule look
defensible for an afternoon, because each wrong verdict was locally coherent
with its own evidence.

**What actually defends against both.** Three things, in order of strength.
First, an external check that the model cannot edit: `ground_truth.json` for the
public set, and a deterministic rule for the private one. Second, keep the
numbers away from the model. A model that cannot recompute a ratio cannot drift
from it, and a rule that is executed in SQL or Python has no mood. Third, make
the failure visible: the run prints the balanced score, each row lists the
expected and predicted verdict, and the fallback path is written down instead of
hidden in an exception. The uncomfortable lesson is that the agent's confidence
and its correctness are produced by completely different parts of the pipeline,
and only one of them is the model.
