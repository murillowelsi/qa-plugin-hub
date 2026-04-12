# ISTQB Testing in Agile Teams

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    AGILE TESTING MINDSET SHIFT                          │
└─────────────────────────────────────────────────────────────────────────┘

  TRADITIONAL                          AGILE
  ───────────                          ─────
  Testing = phase after dev   ───►    Testing = continuous activity
  QA team = separate silo     ───►    QA = embedded in the team
  Big test plan upfront       ───►    Lightweight, evolving plan
  Defects found late          ───►    Defects prevented early
  Sequential phases           ───►    Parallel, iterative cycles
```

---

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     AGILE TESTING QUADRANTS (Brian Marick)              │
└─────────────────────────────────────────────────────────────────────────┘

                     SUPPORT DEVELOPMENT
                            ▲
                            │
          ┌─────────────────┼───────────────────┐
          │   Q2            │      Q1           │
          │  ─────          │     ─────         │
          │  • Functional   │  • Unit Tests     │
          │  • Story Tests  │  • Component      │
  BUSINESS│  • Prototypes   │    Tests          │ TECHNOLOGY
  FACING  │  • Simulations  │  • TDD / BDD      │ FACING
          ├─────────────────┼───────────────────┤
          │   Q3            │      Q4           │
          │  ─────          │     ─────         │
          │  • Exploratory  │  • Performance    │
          │  • Usability    │  • Security       │
          │  • UAT          │  • Load / Stress  │
          │  • Alpha/Beta   │  • Reliability    │
          └─────────────────┼───────────────────┘
                            │
                            ▼
                    CRITIQUE PRODUCT
```

---

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SPRINT TESTING CYCLE                                 │
└─────────────────────────────────────────────────────────────────────────┘

  SPRINT START                                              SPRINT END
      │                                                          │
      ▼                                                          ▼
  ┌───────────┐   ┌───────────┐   ┌───────────┐   ┌───────────┐
  │  SPRINT   │   │    DEV    │   │  TESTING  │   │  SPRINT   │
  │ PLANNING  │──►│  + TEST   │──►│ COMPLETE  │──►│  REVIEW   │
  │           │   │  (daily)  │   │           │   │  + RETRO  │
  └───────────┘   └─────┬─────┘   └───────────┘   └───────────┘
                        │
              ┌─────────┴──────────┐
              │                    │
              ▼                    ▼
        ┌──────────┐         ┌──────────┐
        │  DEV     │         │  TESTER  │
        │  writes  │◄───────►│  writes  │
        │  code    │  collab │  tests   │
        └──────────┘         └──────────┘
              │                    │
              ▼                    ▼
        Unit Tests           Acceptance Tests
        (automated)          (BDD / manual)
```

---

```
┌─────────────────────────────────────────────────────────────────────────┐
│              THE WHOLE TEAM APPROACH (ISTQB Agile Tester)               │
└─────────────────────────────────────────────────────────────────────────┘

                         ┌─────────────────┐
                         │   PRODUCT OWNER │
                         │  (defines ACs)  │
                         └────────┬────────┘
                                  │ User Stories
                                  │ + Acceptance Criteria
                 ┌────────────────┼──────────────────┐
                 ▼                ▼                   ▼
          ┌────────────┐  ┌────────────┐   ┌────────────────┐
          │ DEVELOPERS │  │  TESTERS   │   │  SCRUM MASTER  │
          │            │  │            │   │                │
          │ • Write    │  │ • Story    │   │ • Removes      │
          │   unit     │  │   analysis │   │   blockers     │
          │   tests    │  │ • Test     │   │ • Facilitates  │
          │ • Fix bugs │  │   design   │   │   ceremonies   │
          │ • Support  │  │ • Explore  │   │ • Tracks       │
          │   testing  │  │ • Report   │   │   quality      │
          └────────────┘  └────────────┘   └────────────────┘
                 │                │
                 └───────┬────────┘
                         ▼
                 ┌───────────────┐
                 │  SHARED GOAL  │
                 │  Quality is   │
                 │  everyone's   │
                 │ responsibility│
                 └───────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────────────┐
│              AGILE TESTING ACTIVITIES (mapped to ISTQB process)         │
└─────────────────────────────────────────────────────────────────────────┘

  ISTQB ACTIVITY         AGILE EQUIVALENT
  ───────────────        ────────────────────────────────────────────────
  Test Planning    ──►   Sprint Planning + Definition of Ready (DoR)
  Test Monitoring  ──►   Daily Standup + Sprint burndown + CI dashboards
  Test Analysis    ──►   Backlog Refinement + 3 Amigos session
  Test Design      ──►   BDD scenarios (Given/When/Then) + Test cases
  Test Implement.  ──►   Automate in CI pipeline (unit/integration/e2e)
  Test Execution   ──►   Continuous testing (every commit triggers CI)
  Test Completion  ──►   Sprint Review + Retrospective + DoD check
```

---

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  3 AMIGOS SESSION (key agile practice)                  │
└─────────────────────────────────────────────────────────────────────────┘

       PRODUCT OWNER          DEVELOPER            TESTER
       ─────────────          ─────────            ──────
      "What problem      "How will I build    "How can this
       are we solving?"   this?"               go wrong?"
             │                  │                   │
             └──────────────────┼───────────────────┘
                                ▼
                    ┌────────────────────────┐
                    │   SHARED UNDERSTANDING │
                    │   of the User Story    │
                    │                        │
                    │  ► Clear ACs           │
                    │  ► Edge cases caught   │
                    │  ► No surprises        │
                    └────────────────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────────────┐
│              DEFINITION OF READY vs DEFINITION OF DONE                  │
└─────────────────────────────────────────────────────────────────────────┘

  DoR (Story enters Sprint)        DoD (Story is complete)
  ─────────────────────────        ───────────────────────
  □ Story is written               □ Code reviewed
  □ ACs are clear & testable       □ Unit tests written & passing
  □ Dependencies identified        □ Integration tests passing
  □ Story is estimated             □ Exploratory testing done
  □ Design/mockups ready           □ Regression suite passing
  □ 3 Amigos completed             □ No open critical defects
                                   □ Deployed to staging
                                   □ PO accepted the story
```

---

## Key ISTQB Agile Tester Takeaways

| Concept | Description |
|---|---|
| Shift-Left | Test early, test often — catch bugs before they're expensive |
| CI/CD | Every commit triggers automated tests |
| TDD | Write test → watch it fail → write code → pass |
| BDD | Collaboration via Given/When/Then scenarios |
| Exploratory | Charter-based sessions during the sprint |
| Regression | Automated suite grows sprint over sprint |
