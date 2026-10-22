# Day 1: Software Engineering and the SDLC

*100 Days of Software Engineering, Module: Foundations*

**Time:** about 3 to 4 hours
**You will build:** a requirements-to-release plan for a small app idea, in a GitHub repo with a README and an issue list.
**You should know:** nothing about programming yet. Curiosity and patience are enough.

---

## 1. The big idea

Programming is writing instructions for a computer. Software engineering is deciding what instructions to write, why, in what order, and how to keep them working after the first version ships.

Most beginners think the job is typing code. Professionals spend much of their time on everything around the code: understanding what users need, choosing a design that others can read, testing changes safely, releasing them, and fixing them when reality surprises the team.

The software development life cycle (SDLC) is the name for that whole process. You will meet several models of it. The details differ, but the stages stay the same.

Start with the [Wikipedia article on the software development process](https://en.wikipedia.org/wiki/Software_development_process) and read the introduction and the section on the waterfall and agile models.

---

## 2. The six stages of the SDLC

| Stage | Question it answers | Typical output |
|---|---|---|
| Requirements | What problem are we solving, and for whom? | User stories, acceptance criteria |
| Design / architecture | How will the parts fit together? | Diagrams, data models, interface definitions |
| Implementation | How do we write it? | Code, commits, pull requests |
| Testing | How do we know it works? | Automated tests, manual checks, bug reports |
| Deployment | How does it reach users? | Release, configuration, rollout plan |
| Maintenance | How does it keep working? | Bug fixes, updates, monitoring, technical debt cleanup |

These stages do not happen once in a straight line. Requirements change after testing. Design gets revised after implementation shows a problem. Good teams loop through the stages in small cycles.

---

## 3. Two models: waterfall and iterative

**Waterfall** runs the stages in sequence. You finish requirements, then design, then build, then test. It works when requirements are stable and the cost of change is high, such as building a bridge or a satellite.

**Iterative and agile** work in short cycles. You build a small working piece, show it to users, learn, and repeat. It works when requirements are uncertain, which describes most software.

Here is a simple way to see the difference.

```
Waterfall:   Requirements ──▶ Design ──▶ Build ──▶ Test ──▶ Release
                                                             (one big delivery)

Iterative:   Plan ─▶ Build ─▶ Test ─▶ Release ─▶ Learn ─┐
                 ▲                                        │
                 └────────────── repeat ──────────────────┘
                                 (small deliveries)
```

Neither model is "correct." Your job as an engineer is to know which situation you are in and to avoid the traps of each. Waterfall traps teams into building the wrong thing for a long time. Iterative traps them into making no architectural decisions at all.

**Reference images to find:** search "waterfall vs agile diagram" and pick the one from a source that explains its assumptions. Notice whether the author talks about feedback.

**Video (about 15 minutes):** search "Software Development Life Cycle explained" on YouTube and pick a video from a channel with clear diagrams. freeCodeCamp.org and Simplilearn both publish options. Watch the first half for the stages, then compare with the table above.

---

## 4. Engineering tradeoffs and technical debt

Every decision trades one thing for another.

- Speed of delivery against quality of design.
- Flexibility against simplicity.
- A feature now against maintenance cost later.

**Technical debt** is the cost of shortcuts. A quick fix that saves a day today can cost a week in six months, because the next change has to work around it. Debt is not always bad. A deliberate shortcut for a demo is reasonable if you know you took it and you write it down.

Example: a team stores user passwords in plain text to ship a prototype faster. The shortcut saved a few hours. It created a security risk that now affects every user. The honest version of this decision records the shortcut, the reason, and the plan to fix it.

**Video (about 10 minutes):** search "technical debt explained" on YouTube and watch one short explainer. Then write down one example of debt you have seen in an app you use.

---

## 5. Product thinking: the question before the code

A piece of software is only useful if someone needs it. Before you write a line of code, you should be able to answer:

1. Who is the user?
2. What problem do they have today?
3. How do they solve it now, and what is painful about that?
4. What is the smallest version that would help them?
5. How will you know if it worked?

Write these answers down for your own project. You will reuse them on Day 96 when you plan your capstone.

---

## 6. Hands-on exercise: map an app you already use (about 60 minutes)

Pick a simple app you use every day, such as a notes app, a food delivery app, or a banking app. Without touching its code, write the following:

1. **Requirements:** list five things the app must do for its users. Write each as "a user can ...".
2. **Architecture guess:** draw the main parts you think exist. Examples: a mobile client, a server, a database, a payment service, an email service. Label each one with a guess and mark it "guess."
3. **Testing:** list three things you would test first if you built a clone.
4. **Deployment:** how would users get updates? App store, web page, or both?
5. **Maintenance:** name one thing that would break often over time.

Then check your guesses against the company's public engineering blog or documentation, if it has one. Note what you got right and wrong.

---

## 7. Project: plan a small app from requirements to release

Choose a small app idea. Good examples: a habit tracker, a bookmark manager, a study-timer, or a tool that converts recipes into shopping lists. Keep it small enough to build in a few weeks.

**Your deliverables:**

1. **Problem statement** in two sentences: the user, the problem, the current workaround.
2. **User stories** (at least five) in the form "As a ___, I want ___, so that ___." Add acceptance criteria to each: two or three checkable statements.
3. **Architecture sketch** with the major parts and how data moves between them. A hand-drawn photo is fine.
4. **Test plan:** what you will test automatically, and what you will check by hand.
5. **Release plan:** how the first version reaches a user, and how you will get feedback.
6. **Issue list:** turn your user stories into GitHub issues, one per story.

**Repository setup (GitHub):**

```
<your-app>/
├── README.md              # problem statement, how to run (later), roadmap
├── docs/
│   ├── requirements.md    # user stories and acceptance criteria
│   ├── architecture.png   # your sketch
│   └── test-plan.md
└── .gitignore
```

Create the repository on GitHub, add the files, and make your first commit with a clear message such as "Day 1: problem statement and requirements."

**Done when:**

- The repo exists on GitHub with at least three commits.
- Every user story has at least two acceptance criteria.
- The README explains the problem in two sentences that a non-programmer can understand.
- Your issue list has one issue per user story.

---

## 8. Check your understanding

1. What is the difference between programming and software engineering?
2. Name the six stages of the SDLC in order.
3. Why does waterfall struggle when requirements change?
4. Give one example of technical debt that was a sensible decision.
5. What is the difference between a functional requirement and an acceptance criterion?
6. Why should a release plan include how you will get feedback?

**Answers (check after you try):**

1. Programming is writing instructions. Software engineering also covers requirements, design, testing, release and maintenance, and the tradeoffs among them.
2. Requirements, design, implementation, testing, deployment, maintenance.
3. Waterfall finishes each stage before the next starts. A change discovered late is expensive to absorb, because earlier stages are already locked in.
4. A prototype that skipped a feature for a demo, written in the README as a known gap, with a plan to fix it.
5. A requirement says what the software must do. An acceptance criterion is a specific, testable statement that proves a requirement is met.
6. Without feedback you cannot tell whether the release solved the problem, and you cannot decide what to build next.

---

## 9. Sources to read and watch

**Read:**
- [Wikipedia: Software development process](https://en.wikipedia.org/wiki/Software_development_process)
- [Wikipedia: Waterfall model](https://en.wikipedia.org/wiki/Waterfall_model)
- [Wikipedia: Agile software development](https://en.wikipedia.org/wiki/Agile_software_development)
- [Wikipedia: Technical debt](https://en.wikipedia.org/wiki/Technical_debt)
- [GeeksforGeeks: Software Development Life Cycle (SDLC)](https://www.geeksforgeeks.org/software-engineering-sdlc-software-development-life-cycle/)
- [roadmap.sh: Backend and frontend roadmaps](https://roadmap.sh/) (overview of where the series is going)
- [GitHub Docs: Quickstart for GitHub](https://docs.github.com/en/get-started/start-your-journey/hello-world) (you will use this on your project today)

**Watch:**
- [CS50's Introduction to Computer Science](https://cs50.harvard.edu/) (watch the first lecture for the overall shape of programming)
- [freeCodeCamp.org](https://www.youtube.com/@freecodecamp): search "software development life cycle" and watch one introduction.
- [Computerphile](https://www.youtube.com/@Computerphile): search "software engineering" for a short explainer.

**Practice:**
- [Git documentation: Getting Started](https://git-scm.com/book/en/v2/Getting-Started-About-Version-Control) (read the first section; you will start the Git track on Day 6)

**Note on links:** open each one once. If a page has moved, search its title.

---

## 10. Tomorrow

Day 2, **How Computers Execute Programs**, explains what happens between the code you write and the result on screen: the CPU, memory, compilation and interpretation. You will need your app idea from today, because tomorrow you will trace how a simple feature from it would run.
