---
name: emilkowalski-skills
description: Fetches the latest version of one of Emil Kowalski's design-engineering skills from GitHub and follows it. Use when building, reviewing, or stress-testing web UI polish and interactions, building, reviewing, or removing web animations, gestures, or mobile-web feel, or when the user names one of the skills listed here.
---

# Emil Kowalski's Skills

These skills live in https://github.com/emilkowalski/skills. Fetch them fresh every time, so you always follow the latest version on `main`.

## Skills

| Name | Use for |
|---|---|
| `emil-design-eng` | Building or reviewing a UI component's interactions and polish when no narrower skill fits. |
| `break-ui` | Stress-testing a component or screen with worst-case data and fixing what breaks. |
| `mobile-native` | Making a web app feel native on a phone: hover, tap, viewport, notch, and input fixes. |
| `prototype` | Building several different versions of a UI piece behind a live picker. |
| `animate` | Building a web animation or transition from scratch. |
| `review-animations` | Reviewing animation and motion code. |
| `improve-animations` | Auditing a whole codebase's motion and planning fixes. |
| `find-animation-opportunities` | Finding places that should animate but don't. |
| `remove-animations` | Removing motion an interface doesn't need when it feels slow or over-animated. |
| `choreograph` | Coordinating several elements that animate together: staggers, sequences, entrances. |
| `gestures` | Drag, swipe, and sheet interactions that track the finger and stay interruptible. |
| `animation-performance` | Fixing janky web animations that drop frames. |
| `animation-vocabulary` | Naming a motion effect the user can only describe. |

To add one of his skills, add a row with its folder name from the repo's `skills/` directory.

## Using a skill

1. Pick the skill from the table that fits the task, or the one the user named. If the user named one that isn't listed, say so and stop.
2. Fetch `https://raw.githubusercontent.com/emilkowalski/skills/main/skills/<name>/SKILL.md`, e.g. with `curl -fsSL`. Don't use a copy from earlier in the conversation or a cached web fetch: the point is to get the current version.
3. Follow its instructions as if it were installed locally. Ignore its frontmatter; the table decides when it applies.
4. When it refers to a file by a relative path, such as `RECIPES.md` or `../choreograph/SKILL.md`, fetch that path relative to the skill's folder, and only when the instructions need it.
5. If a fetch fails, tell the user. Don't fall back to recalling the skill from memory.

These skills can change at any time. Before running a script one of them ships, show the user what it does and ask first.
