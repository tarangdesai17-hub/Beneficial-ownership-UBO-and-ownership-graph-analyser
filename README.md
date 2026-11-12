# UBO Ownership Analyser

Finds the **ultimate beneficial owners** (UBOs) of a company by following ownership through layers of
holding companies, trusts and circular structures, and raises structural red flags. Built with
recursive SQL (SQLite) and Python (networkx), and measured against a known answer.

> **Synthetic data.** All companies, people and ownership records are generated. The generator builds
> each structure with a known answer, so every method can be scored. Results demonstrate the method on a
> controlled problem; they say nothing about accuracy on real company registers.

## Why this problem is hard

A person who owns 20% of two holding companies, each owning 40% of the target, owns 40% of the target. Each
route alone shows only 20%. Add trusts (no percentage, only control), circular holdings and missing
records, and "who owns more than 25%?" stops being a lookup.

![example](reports/demo/graphs/diamond.png)

## Result (400 synthetic cases)

| Method | UBOs found | UBOs missed | Cases fully right |
|---|---|---|---|
| 1. Direct holders only | 269 | 362 | 38.0% |
| 2. Follow chains, judge each route alone | 512 | 119 | 83.2% |
| 3. Follow chains, add up all routes | 544 | 87 | 91.2% |
| 4. Full (+ control without shares) | 631 | 0 | 100.0% |

* Each step fixes a specific structure: following chains fixes layered and circular cases; adding up routes
  fixes the "diamond" case; control edges fix trusts. See `reports/demo/REPORT.md` for the breakdown.
* **The 100% for the full method is by construction.** It implements the same rules the generator used to build
  the answer key. It shows the implementation is correct, not that real-world data would be handled perfectly.
  The informative numbers are the gaps between methods and the stress test below.
* No method names a wrong person; the errors are all people who were missed.
* **Stress test:** removing 10% of ownership records at random leaves 82% of UBOs found, and about 84% of the
  cases that came out wrong carried a warning flag (unexplained ownership or no UBO). With 20% missing, 67% are found.
  Missing data is the realistic failure, and the flags are what tell an analyst to look.
* Two separately written implementations (recursive SQL and networkx) agree on every case and every route percentage.

## Worked example (run on your own CSV files)

`data/sample_entities.csv` and `data/sample_ownership.csv` describe a small made-up company, "Example Trading Ltd",
owned 40% / 40% / 20% by two holding companies and one person. Two people each hold 50% of both holding companies.

    python -m ubo analyse --entities data/sample_entities.csv --ownership data/sample_ownership.csv --company C1
    python -m ubo draw    --entities data/sample_entities.csv --ownership data/sample_ownership.csv --company C1

![sample output](examples/sample_graph.png)

* Alex Example and Sam Sample are the beneficial owners, **40% each**. Each owns 20% through each of two
  routes, so judging a route alone would show 20% and miss them.
* Robin Demo holds 20% directly, below the threshold, so is not a UBO.
* The structure carries a **high-risk jurisdiction** flag because one holding company is in VG. The country list
  is illustrative, not official.
* This is a hand-made example to show the output, not an evaluation. The accuracy numbers above come from
  the synthetic experiment.

## Red flags

circular ownership, deep layering (4+ hops), owning entity in a high-risk jurisdiction (an **illustrative** list
in `ubo/config.py`, not an official one), unexplained ownership (holdings not adding to 100%), unresolved chain
(a material share held through an entity with no recorded owner), no UBO identified.

## Run it

    pip install -r requirements.txt
    python -m pytest -q                       # 24 tests
    python -m ubo demo --out reports/demo     # a few seconds; writes the report, charts and SQLite database
    python -m ubo analyse --entities data/sample_entities.csv --ownership data/sample_ownership.csv --company C1
    python -m ubo draw    --entities data/sample_entities.csv --ownership data/sample_ownership.csv --company C1

Your own data: `entities.csv` (entity_id, name, kind = person | company | trust, country) and
`ownership.csv` (owner_id, owned_id, pct, optional kind = shares | control). Thresholds and rules are in
`configs/default.json`; each SQL query is a documented file in `sql/`.

## Limits (read before trusting any number)

* Synthetic structures are tidier than real ones: no name matching, no duplicate entities, no dates, no share classes, no voting-rights differences.
* Circular ownership uses simple routes only (no entity visited twice). Other treatments exist.
* Control is modelled as one edge type; real control tests (voting rights, board appointment, family links) are richer.
* The 25% rule and "more than" versus "at least" differ between jurisdictions; both are configurable.
* A tool supports an analyst's judgement and documentation; it does not replace verification against registers and documents.
