# engineering-skills

Fifteen Claude skills distilled from first-principles engineering lectures, plus detailed per-video notes.

| Skill | Built from |
|---|---|
| [`backend-first-principles`](backend-first-principles/SKILL.md) | 127 videos: Sriniously (Backend from First Principles), Hussein Nasser, CMU Database Group (Andy Pavlo), MIT 6.824 (Robert Morris), Martin Kleppmann, Arpit Bhayani, ByteByteGo |
| [`data-engineering-first-principles`](data-engineering-first-principles/SKILL.md) | 66 videos: Zach Wilson (DataExpert), Seattle Data Guy, Andreas Kretz, Joe Reis |
| [`ai-engineering-first-principles`](ai-engineering-first-principles/SKILL.md) | 43 talks: Andrej Karpathy, Hamel Husain & Shreya Shankar, Chip Huyen, Jason Liu, Anthropic and AI Engineer conference speakers |
| [`ml-engineering-first-principles`](ml-engineering-first-principles/SKILL.md) | 27 lectures: Andrej Karpathy, Andrew Ng (CS229), Jeremy Howard (fast.ai), Martin Zinkevich, Full Stack Deep Learning, Shreya Shankar, WhyLabs, StatQuest, Stanford CS336 |
| [`data-science-statistics-first-principles`](data-science-statistics-first-principles/SKILL.md) | 32 lectures: Richard McElreath, Brady Neal, Joe Blitzstein (Stat 110), 3Blue1Brown, Ronny Kohavi, StatQuest |
| [`software-craft-first-principles`](software-craft-first-principles/SKILL.md) | 28 talks: John Ousterhout, Rich Hickey, Sandi Metz, Gary Bernhardt, Brandon Rhodes, Martin Fowler, Kent Beck, Michael Feathers, J.B. Rainsberger, Ian Cooper, Raymond Hettinger, Casey Muratori |
| [`systems-thinking-first-principles`](systems-thinking-first-principles/SKILL.md) | 21 talks: Donella Meadows, John Sterman (MIT), Russell Ackoff, Peter Senge, Dave Snowden, Derek Cabrera, Richard Cook, Sidney Dekker, Nancy Leveson, Goldratt, Munger |
| [`geospatial-data-engineering-first-principles`](geospatial-data-engineering-first-principles/SKILL.md) | 23 talks: Paul Ramsey (PostGIS), DuckDB spatial, Kyle Barron (GeoParquet), Overture Maps, Dunnington & Pebesma (S2), Matt Forrest, Josiah Parry, FOSS4G cloud-native talks, GTFS |
| [`entity-resolution-first-principles`](entity-resolution-first-principles/SKILL.md) | 14 talks: Robin Linacre (Splink), Dean Resnick and Thibaudeau (Fellegi-Sunter), Zingg, dedupe, Senzing, Tamr, BM25 fuzzy matching (PyData), Peter Christen |
| [`api-payments-security-first-principles`](api-payments-security-first-principles/SKILL.md) | 17 talks: Feross Aboukhadijeh (Stanford CS253), Corey Ball and InsiderPhD (OWASP API Top 10), OktaDev (OAuth/OIDC), Computerphile, HashiCorp, Arpit Bhayani (idempotency), PCI DSS |
| [`startup-product-first-principles`](startup-product-first-principles/SKILL.md) | 19 talks: Paul Graham, Michael Seibel, Dalton Caldwell, Jared Friedman, Sam Altman, Rob Fitzpatrick, April Dunford, Todd Jackson, Rob Snyder, David Rusenko, Peter Reinhardt |
| [`search-ranking-recsys-first-principles`](search-ranking-recsys-first-principles/SKILL.md) | 17 talks: Eugene Yan, Doug Turnbull, Booking.com, Andrew Ng, Christopher Potts, Haystack LTR panel, BM25 and ranking-metric explainers |
| [`production-ops-first-principles`](production-ops-first-principles/SKILL.md) | 17 talks: Charity Majors, Liz Fong-Jones, Google SRE authors, Dave Farley, Nicole Forsgren (DORA), Richard Cook, Edson Yanaga (zero-downtime migrations) |
| [`frontend-fundamentals-first-principles`](frontend-fundamentals-first-principles/SKILL.md) | 13 talks: Philip Roberts, Lydia Hallie, Kevin Powell, Matt Pocock, Carson Gross (htmx), freeCodeCamp accessibility, Core Web Vitals |
| [`math-for-engineers-first-principles`](math-for-engineers-first-principles/SKILL.md) | 19 lectures: 3Blue1Brown (linear algebra, calculus, neural networks), Gilbert Strang (MIT 18.06/18.065), Joe Blitzstein (Stat 110) |

Each folder has `SKILL.md` (procedure, principles, checklists) and `references/` (one section per
video with mechanisms, numbers, trade-offs and corrections, credited to its teacher).
`references/INDEX.md` lists every video with its line number.

Notes are paraphrased summaries written for personal study; watch the original videos for the full
teaching. Lines marked "(note: …)" and "Relevance to Sam" are additions; "Not covered by the sources" boxes are reviewer background, not from the videos, not the teacher's words.

## Using the notes from a skill

Raw file URLs (private repo, so fetch with an authenticated GitHub tool, e.g. `gh api` or a clone):

```
git clone --depth 1 https://github.com/Biodun-ml/engineering-skills
grep -n -i "idempotency" engineering-skills/*/references/*.md
```
