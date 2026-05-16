# Project Setup Overview (Sprint 1)

## 1. Guidelines (Foundation – MUST be done first)
**Responsible:** Lev, Valentina

- Code standards & versions (libs.versions.toml)
- TDD approach 
- Dependency & plugin definitions
- Branching strategy (feature branches only)
- Workflow:  
  → pull → branch → develop → push → PR → merge → pull

⚠️ Everyone depends on this → must be defined first

---

## 2. Docker Setup (Environment)
**Responsible:** Moritz

- Same environment for everyone
- Avoid “works on my machine”
- Define ports + services

⚠️ Blocks WebSocket setup (ports must be open)

---

## 3. WebSocket (Core Communication)
**Responsible:** Lev, Valentina

- Connect backend ↔ frontend
- Real-time communication

⚠️ Depends on Docker configuration  
Frontend/backend dev depends on this

---

## 4. CI Pipeline
**Responsible:** Lea Kutschera

- Auto build & test on push
- Ensures code works before merge

👉 Can start early, but useful after basic setup

---

## 5. SonarCloud + Quality Gates
**Responsible:** Fabian, Moritz

- Code quality & security checks
- Test coverage

👉 Integrated into CI pipeline

---

## 6. User Stories & UI Sketches (for upcoming task assignment)

⚠️ Not part of technical setup, but important for planning next tasks

### User Stories (feature perspective)
**Responsible:** Lea Kanduth (additional 1 person e.g. Lev)

- Define what the player should be able to do  
  → e.g. “As a player, I want to roll the dice so that I can move on the board.”  
- https://github.com/solid/user-stories

---

### UI Sketches (interaction perspective)
**Responsible:** Ivona, Lea Kanduth

- Define how this is visualized in the app (as simple as possible)
- Important: be realistic and do not overshoot
- Align with user stories to define needed screens  
  → e.g. screen with dice button, cards, dice, next player indicator

---

### 👉 These are used to:
- understand the game logic clearly
- break features into tasks
- assign work in the team for the next sprint

---

## Workflow Rules (IMPORTANT)

- Always work on feature branches
- Never commit directly to main
- Always:  
  → pull → develop → push → PR → merge

👉 Main must always be working + passing tests

---

## Dependencies (WHO WAITS FOR WHOM)

- Everyone waits for → Guidelines
- WebSocket waits for → Docker
- Frontend/backend depend on → WebSocket
- SonarCloud depends on → CI

---

## Goal for Sprint 1 (from SE2)

- Working GitHub project + board
- CI + SonarCloud running
- Basic WebSocket communication
- First game logic implemented
- Tests working

---

## QUICK OVERVIEW

### A. Planning
- Guidelines
- Choose game (Machi KORO)
- Define user stories
- Sketch UI

---

### B. Setup
- GitHub board/issues
- CI pipeline
- SonarCloud
- Docker
- WebSocket base

---

### C. Build
- Break user stories into tasks
- Assign people
- Implement
- Test
- Merge via PR
