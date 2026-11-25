# engineering-skills

Seven Claude skills distilled from first-principles engineering lectures, plus detailed per-video notes.

| Skill | Built from |
|---|---|
| [`backend-first-principles`](backend-first-principles/SKILL.md) | 127 videos: Sriniously (Backend from First Principles), Hussein Nasser, CMU Database Group (Andy Pavlo), MIT 6.824 (Robert Morris), Martin Kleppmann, Arpit Bhayani, ByteByteGo |
| [`data-engineering-first-principles`](data-engineering-first-principles/SKILL.md) | 66 videos: Zach Wilson (DataExpert), Seattle Data Guy, Andreas Kretz, Joe Reis |
| [`ai-engineering-first-principles`](ai-engineering-first-principles/SKILL.md) | 43 talks: Andrej Karpathy, Hamel Husain & Shreya Shankar, Chip Huyen, Jason Liu, Anthropic and AI Engineer conference speakers |
| [`ml-engineering-first-principles`](ml-engineering-first-principles/SKILL.md) | 27 lectures: Andrej Karpathy, Andrew Ng (CS229), Jeremy Howard (fast.ai), Martin Zinkevich, Full Stack Deep Learning, Shreya Shankar, WhyLabs, StatQuest, Stanford CS336 |
| [`data-science-statistics-first-principles`](data-science-statistics-first-principles/SKILL.md) | 32 lectures: Richard McElreath, Brady Neal, Joe Blitzstein (Stat 110), 3Blue1Brown, Ronny Kohavi, StatQuest |
| [`software-craft-first-principles`](software-craft-first-principles/SKILL.md) | 28 talks: John Ousterhout, Rich Hickey, Sandi Metz, Gary Bernhardt, Brandon Rhodes, Martin Fowler, Kent Beck, Michael Feathers, J.B. Rainsberger, Ian Cooper, Raymond Hettinger, Casey Muratori |
| [`systems-thinking-first-principles`](systems-thinking-first-principles/SKILL.md) | 21 talks: Donella Meadows, John Sterman (MIT), Russell Ackoff, Peter Senge, Dave Snowden, Derek Cabrera, Richard Cook, Sidney Dekker, Nancy Leveson, Goldratt, Munger |

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
