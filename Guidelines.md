# Guidelines - Team Development Workflow & Standards

## 1. Purpose
This document defines the common development rules for the project so that all team members work with the same standards, versions, workflow, and quality expectations.

## 2. Code standards & versions
- All versions are managed centrally in libs.versions.toml 
- No direct version numbers in build.gradle.kts
- Use aliases from libs
- Existing versions should only be changed if really necessary
- Code should follow consistent formatting and naming conventions
- Keep implementations simple and readable

## 3. TDD approach
- Follow Test-Driven Development where possible
- Write or think about tests before implementing logic
- Core game logic should always be testable
- Prefer small, incremental steps:
  - write failing test
  - implement minimal solution
  - refactor safely

**Helpful resource:**  
Fireship TDD video - https://www.youtube.com/watch?v=Jv2uxzhPFl4&ab_channel=Fireship

## 4. Dependencies & plugins
- New dependencies/plugins must first be added to libs.versions.toml
- Only then can they be referenced in Gradle
- Avoid unnecessary libraries
- Discuss major additions with the team before using them

## 5. Branching strategy
- Never work directly on main
- Each task/story gets its own feature branch
- Use pull requests for merging into main
- main must always stay runnable

**Helpful resource:**  
Atlassian Feature Branch Workflow - https://www.atlassian.com/git/tutorials/comparing-workflows/feature-branch-workflow

## 6. Commit strategy
- Use clear commit messages
- Follow Conventional Commits where possible

**Example:**
- feat: add dice roll logic
- test: add player movement tests
- fix: correct card income calculation

**Helpful resource:**  
Conventional Commits - https://www.conventionalcommits.org/en/v1.0.0/

## 7. Workflow
- Pull latest changes first
- Create your branch
- Develop in small commits
- Push regularly
- Open PR
- Merge only after review / checks pass
- Pull again before continuing new work

**Flow:**  
pull → branch → develop → push → PR → merge → pull

## 8. Quality expectations
- Code should compile before pushing
- Tests should pass before merging
- Keep main stable and working
- SonarCloud / CI checks must not be ignored

## 9. User Stories & Definition of Done
- Work is based on User Stories
- Each story must include:
  - clear description
  - acceptance criteria
- Stories are managed in the GitHub backlog
- A task is considered done only if:
  - implementation is complete
  - tests are written and passing
  - code is reviewed
  - CI / SonarCloud checks pass
  - merged into main
