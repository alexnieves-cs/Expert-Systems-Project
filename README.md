# Job Qualification Expert System

A rule-based expert system that asks an applicant a set of questions, then reports:

1. The positions they **qualify** for, with notes about any missing *desired* skills.
2. The positions they **don't qualify** for, listing **every** failed condition for each one.

It is a single static page (`index.html`) written in plain HTML, CSS, and JavaScript. It has no frameworks, no build step, and no dependencies.

## Live demo

- **App:** https://alexnieves-cs.github.io/Expert-Systems-Project/
- **Tests:** https://alexnieves-cs.github.io/Expert-Systems-Project/tests.html
- **Source:** https://github.com/alexnieves-cs/Expert-Systems-Project

## Running it

**GitHub Pages:** push the repo, then go to *Settings → Pages → Deploy from a branch → `main` / root*. The app is served at `https://<user>.github.io/<repo>/`, and the tests at `.../tests.html`.

**Locally:**

```sh
python3 -m http.server 8000
# then open http://localhost:8000/ and http://localhost:8000/tests.html
```

You can also open `index.html` directly (double-click it). `tests.html` must be served over HTTP, though, as above or on GitHub Pages. It loads the engine from `index.html` in a hidden iframe, and browsers block that for `file://` pages.

## Files

| File | Purpose |
|---|---|
| `index.html` | The app: the knowledge base, the inference engine, and the UI |
| `tests.html` | Test runner: 15 applicant profiles plus 27 input-validation cases, run through the same engine |
| `README.md` | This document |

## How the rules engine works

`index.html` contains two scripts:

- `<script id="engine">` holds the **knowledge base** and the **inference engine**. It is pure logic with no DOM access, and it is exposed as `window.ExpertEngine`.
- The second script is the **UI**. It validates input, turns the answers into *facts*, calls the engine, and renders the results.

### Facts

The answers become a facts object:

```js
{
  degreeLevel: 'none' | 'bachelors' | 'masters' | 'phd',
  degreeField: 'cs' | 'other' | null,      // null when degreeLevel is 'none'
  pythonCourse, seCourse, agileCourse,     // booleans
  pythonYears, dataDevYears, dataArchYears,
  expertSystemsYears, agileYears, pmYears, // integers 0–60
  usedGit, pmiCert                         // booleans
}
```

### Rules as data

The knowledge base is a flat `RULES` array. Each rule is one condition for one position:

```js
{ id: 'pe-python-3', position: 'pe', type: 'needed',
  label: '3+ years of Python development',
  test: { kind: 'min', fact: 'pythonYears', min: 3 } }
```

- **`type`**
  - `needed` and `qualification` are pass/fail. Failing either one disqualifies the applicant.
  - `desired` never disqualifies anyone. When it fails, it is reported as a note.
- **`test`** is a declarative condition. The engine understands three kinds:
  - `yes`: the fact must be `true`
  - `min`: the fact must be `>= min`
  - `degree`: the degree rank must be `>= minLevel` **and** the field must match

To add a position or change a requirement, you edit data only. The engine code doesn't change.

### Inference

`evaluate(facts)` runs every rule for every position. It records a pass or fail for each rule, with a human-readable explanation such as *"3+ years of Python development: you have 2 years (1 short)"*. Then it concludes:

- **qualified** ⇔ no `needed` or `qualification` rule failed
- `failed`: every failed `needed` or `qualification` rule (all of them are listed, not just the first)
- `missingDesired`: every failed `desired` rule

Each result card has a **"How this was decided"** section. It shows the full trace of rules checked, so every conclusion can be explained.

## Positions encoded

| Position | Needed | Desired | Qualification |
|---|---|---|---|
| Entry-Level Python Engineer | Python coursework; Software Engineering coursework | Agile course | Bachelor in CS |
| Python Engineer | 3+ yrs Python; 1+ yr data development; Agile project experience | Used Git | Bachelor in CS |
| Project Manager | 3+ yrs managing software projects; 2+ yrs Agile projects | none | PMI Lean Project Management Certification |
| Senior Knowledge Engineer | 4+ yrs Python; 2+ yrs expert systems; 2+ yrs data architecture; 2+ yrs data development | none | Masters in CS |

## Assumptions

- **Thresholds are inclusive.** "3+ years" means `>= 3`, so exactly 3 passes and 2 fails.
- **Degree hierarchy:** None < Bachelors < Masters < PhD. A higher degree satisfies a lower requirement. A Masters satisfies "Bachelor in CS", and a PhD satisfies both "Bachelor in CS" and "Masters in CS".
- **The degree field must be Computer Science** for any degree qualification. A Masters in "Other" fails "Bachelor in CS".
- **The degree qualification is one condition.** When it fails, its explanation names what is wrong: the level is too low, the field is not CS, or there is no degree.
- **"Experience in Agile projects"** (Python Engineer) means at least 1 year on Agile projects. The same "years on Agile projects" answer is used for the Project Manager's 2+ year rule.
- **"2+ years data architecture and data development"** (Senior Knowledge Engineer) is two separate conditions: 2+ years of data architecture **and** 2+ years of data development.
- **Desired skills never disqualify anyone.** For a qualified applicant they appear as *"Meets requirements, missing desired skill: Git"*. For an unqualified applicant they appear as an extra note.
- **Project Manager needs no degree**, because none is listed.

## Input validation

- **Degree level** is typed text and must be one of `None`, `Bachelors`, `Masters`, or `PhD`. Case and extra spaces are ignored. Anything else is rejected. Common abbreviations get a targeted message; for example, `BS` gives *"'BS' isn't accepted. Enter it as 'Bachelors'…"*, and the same goes for `MS` and `Ph.D.`.
- **Degree field** is typed text and must be `Computer Science` or `Other` (case-insensitive). Abbreviations like `CS` or `Comp Sci` are rejected with a message asking for "Computer Science". If the degree level is `None`, this field is disabled and not required.
- **Years fields** accept whole numbers from 0 to 60 only. Blank values, negatives, decimals (`3.5`), words, and exponent notation (`1e1`) are rejected, each with a specific message. Surrounding spaces are ignored.
- **Yes/no questions** are radio buttons with no default, so the applicant has to choose.
- **Errors show inline** next to each field:
  - Before the first submit, a text field is validated when you leave it.
  - After you press *See my results*, every field is validated. The first invalid field gets focus, and a count of the remaining errors appears next to the button.
- **Results appear only when every field is valid.** After the first submit, results update live as you edit. If any field becomes invalid, the results are hidden again.

## Tests

Open `tests.html` over HTTP. It shows a pass/fail banner and a card for each test, and each card includes the engine's full reasoning trace.

**Applicant profiles (15).** Each test states the exact set of failed rule IDs it expects per position, not just qualified or not. The profiles cover:

- **Qualifies for everything:** one profile exactly at every threshold, and one PhD profile well above every threshold.
- **A blank slate:** every failed condition is listed.
- **Each position at its thresholds:** Python Engineer, Project Manager, and Senior Knowledge Engineer each have one profile *exactly at* the thresholds and one *one below* each threshold.
- **Degree cases:**
  - A Masters holder applying for a Bachelor role
  - A Bachelors holder applying for a Masters role
  - A non-CS degree
  - A PhD satisfying the Masters requirement
- **Desired skills:** a missing desired skill doesn't disqualify, and desired skills alone don't qualify.

**Input validation (27).** Accepted and rejected inputs for every validator, including `BS`, `MS`, `CS`, `61`, `-1`, `3.5`, and blank.
