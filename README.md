# 100 Ultralearning

A self-study curriculum in three tracks, one lesson per day for 100 days:

- **AI Engineering**
- **Robotics Engineering**
- **Software Engineering**

Each day is a self-contained tutorial with concepts built from first principles, a hands-on exercise, and a project you commit to GitHub. The goal is working skill you can show an employer or a client, not just familiarity with terms.

## Tracks

| Track | Index | Modules | Written so far |
|---|---|---|---|
| AI Engineering | [100_Days_AI_Engineering.md](AI/100_Days_AI_Engineering.md) | 12: foundations, math, data engineering, classical ML, deep learning, computer vision, NLP, LLMs, agents, MLOps/LLMOps, edge and robotics AI, capstone | Days 1–16 (Modules 1–2) |
| Robotics Engineering | [100_Days_Robotics_Engineering.md](ROBOTICS/100_Days_Robotics_Engineering.md) | 14: foundations, electronics, embedded systems, sensors and actuators, mechanical design and CAD, PCB design, robot math, kinematics, control, ROS 2, simulation, autonomy, modern robotics, capstone | Days 1–15 (Modules 1–2) |
| Software Engineering | [100_Days_Software_Engineering.md](SOFTWARE/100_Days_Software_Engineering.md) | Foundations, Git and collaboration, Python, C++, data structures and algorithms, web, React, backend and databases, full stack and mobile, systems and cloud, capstone | Days 1–8 (Foundations and Git) |

Each index file lists every day with its study topics, its output or exercise, and a link to the tutorial.

## Repository layout

```
100 ULTRALEARNING/
├── AI/
│   ├── 100_Days_AI_Engineering.md           # AI track index
│   └── 100 Days AI/
│       ├── Module-01-AI-Foundations/
│       └── Module-02-Mathematics-for-AI/
├── ROBOTICS/
│   ├── 100_Days_Robotics_Engineering.md     # Robotics track index
│   └── 100 Days Robotics/
│       ├── Module-01-Robotics-Engineering-Foundations/
│       └── Module-02-Electronics-for-Robotics/
├── SOFTWARE/
│   ├── 100_Days_Software_Engineering.md     # Software track index
│   └── 100 Days Software/
│       ├── Module-01-Foundations/
│       └── Module-02-Git-and-Collaboration/
└── Day-Generator-Prompt.md                  # Prompt for writing new day lessons
```

## What each day contains

1. Title, time estimate, level, outcomes, equipment, prerequisites, and a table of contents
2. The big idea: why the topic matters on the job
3. Glossary of new terms
4. Concept deep dives, with analogies, worked examples, common mistakes, and real-world use
5. Hands-on exercises (about 60 minutes)
6. A project with a repo layout, starter steps, and a "done when" list
7. Check-your-understanding questions with answers
8. Sources: official docs, tutorials, videos, and one practice resource
9. A short preview of the next day

Days in the software and robotics tracks use the same structure, with tools suited to each field (CI/CD and tests for software, ROS 2, simulators, and microcontrollers for robotics).

## How to use it

1. Open the index for the track you are following.
2. Work through the days in order. Each day builds on the one before it.
3. Do the exercises and the project yourself before moving on.
4. Commit the project to GitHub. The index and the lesson each say what evidence to commit (code, tests, results, a README).

The curriculum ends with a capstone in Days 99–100 and a career section (Day 101) covering job direction, portfolio building, and first paid work.

## Writing a new day

[Day-Generator-Prompt.md](Day-Generator-Prompt.md) is a reusable prompt. Replace `{SERIES}`, `{DAY_NUMBER}`, and `{NOTION_OUTLINE}` with the track, day number, and that day's outline, then save the output as `Day-NN-<slug>.md` in the matching module folder. Then add the day to the track's index.

File names follow the track prefix:

- AI: `Day-01-What-Is-Artificial-Intelligence.md`
- Robotics: `Robotics-Day-01-What-Is-Robotics.md`
- Software: `Software-Day-01-Software-Engineering-and-the-SDLC.md`
