---
spec_id: 001-persistent-tasks-with-web-ui
title: Persistent tasks with web UI
status: done
priority: P1
shape: gitops-app
---

# 001 — Persistent tasks with web UI

## Problem

The Task API keeps tasks in memory, so every restart or redeploy of the API loses every task, and the API is the only way to see or record one. People who are not API clients cannot use the taskboard at all, and the web component shows only a starter page. Storing tasks in a Postgres database makes them survive restarts, and a task page in the web UI lets people list and create tasks without calling the API by hand.

## Stories

As a taskboard user, I want the tasks I record to still be there after the system restarts or is redeployed, so that I can rely on the taskboard as the record of work to be done.
**Repos:** taskboard11-api, taskboard11-gitops

As a taskboard user, I want to see all tasks with their due dates on a web page, so that I know what has been recorded without calling the API.
**Repos:** taskboard11-web, taskboard11-api, taskboard11-gitops

As a taskboard user, I want to create a task with a title and an optional due date from the web page, so that I can record work without an API client.
**Repos:** taskboard11-web, taskboard11-api

## Acceptance criteria

- **AC-001.1** Given tasks were created through `POST /tasks`, when the Task API restarts or is redeployed, then `GET /tasks` returns 200 with each of those tasks, with the same `id`, `title` and `due_date`, in creation order.
- **AC-001.2** Given the database cannot be reached, when a client sends `POST /tasks` or `GET /tasks`, then the response is 503, the Task API's readiness check fails, and once the database is back a task sent during the outage is not in `GET /tasks`.
- **AC-001.3** Given tasks A, B and C were created in that order (through the API or the web UI), when a user opens the task page of the web UI, then the page lists the three tasks, A before B before C, each with its title and its due date as `YYYY-MM-DD`, or "No due date" for a task without one.
- **AC-001.4** Given no task exists, when a user opens the task page, then the page shows "No tasks yet" and the create form.
- **AC-001.5** Given the task page is open, when a user enters the title "Buy milk" and the due date 2026-12-01 (or leaves the due date empty) and submits the form, then "Buy milk" with that due date (or "No due date") appears at the end of the list without a manual page reload, the form is cleared, and `GET /tasks` contains the task.
- **AC-001.6** Given the task page is open, when a user submits the form with an empty or whitespace-only title, or a title longer than 200 characters, then the page shows "Title must be 1 to 200 characters", keeps the entered values, and no task is created.
- **AC-001.7** Given the Task API cannot be reached or answers with an error, when a user opens the task page, then the page shows "Tasks could not be loaded. Try again later." instead of the list; and when a user submits the form in that state, then the page shows "Task could not be saved. Try again." and keeps the entered values.

## Non-goals

- Editing, completing or deleting tasks, in the API or in the web UI.
- Filtering, sorting options or pagination of the list.
- Authentication, user accounts and per-user task lists: the web UI and the Task API stay reachable without login, which is accepted for dev only. Authentication is a later spec that must be done before either component is promoted to staging or prod.
- Migrating tasks that exist in the in-memory store at the switch-over: they are lost once, as with any restart today.
- Database backups and restore (a later spec, through the system's `backup:` settings).
- Changes to the Task API's request and response format (paths, fields, validation and status codes stay as they are).
- Live updates of the task page when another person creates a task (a reload shows it).

## Assumptions

- **A1** Tasks are kept in one Postgres database declared as a resource of the taskboard11 system and used by the Task API only; the web UI reads and writes tasks only through the Task API (other option: the web UI reads the database directly; affects AC-001.1, AC-001.3, AC-001.5).
- **A2** When the database is unreachable the Task API answers 503 on the task endpoints and fails its readiness check, and does not fall back to memory (other option: a memory fallback; affects AC-001.2).
- **A3** The task page is the web UI's start page, replacing the starter page (other option: a separate path such as `/tasks`; affects AC-001.3 to AC-001.7).
- **A4** The list shows tasks oldest first, all of them, exactly as `GET /tasks` returns them; a new task appears at the end (other option: newest first; affects AC-001.3, AC-001.5).
- **A5** The form checks the title with the API's rules (1 to 200 characters, at least one non-whitespace character) before sending it, and the due date is picked as a calendar date and may lie in the past (other option: rely on the API's 422 only; affects AC-001.6).
- **A6** Page texts are English and exactly: "No tasks yet", "No due date", "Title must be 1 to 200 characters", "Tasks could not be loaded. Try again later.", "Task could not be saved. Try again." (other option: other wording; affects AC-001.3, AC-001.4, AC-001.6, AC-001.7).
- **A7** The database runs in the cluster through the platform's Postgres resource, in each environment where the Task API runs, today dev only (other option: a managed external database; affects AC-001.1, AC-001.2).

## Open questions

- ~~The web UI is published through the Gateway with no login, and the Task API is exposed without authentication too, so anyone who can reach the web page can read every task and create tasks. Is that acceptable for this spec? (suggested: yes, for dev now; authentication is a later spec before promoting to staging or prod; affects AC-001.3, AC-001.5, new criterion if no)~~ Answered 2026-11-12: yes, for dev only; folded into the non-goals.

## Changelog

- 2026-11-12 created
- 2026-11-12 amended: open question on access without login answered (yes, dev only; authentication is a later spec needed before staging or prod), folded into the non-goals
- 2026-11-12 approved
- 2026-11-12 building
- 2026-11-12 done
