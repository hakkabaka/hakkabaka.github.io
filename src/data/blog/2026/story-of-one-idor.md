---
title: Finding an IDOR by Learning the App’s Role Model
author: hakkabaka
pubDatetime: 2026-03-17T05:17:19Z
slug: finding-idor-pm-case-study
featured: false
draft: false
tags:
  - idor
  - bug-bounty
  - web-security
# ogImage: "https://example.org/remote-image.png" # remote URL
description: How understanding roles, visibility states, and configuration gates led to discovering an IDOR in a project management system
canonicalURL: https://example.org/my-article-was-already-posted-here
---


## TL;DR

I found an **Insecure Direct Object Reference (IDOR)** in a project management system.

The application had a *“Shared Project”* type, but by default it was supposed to behave like a private project and remain hidden from regular users. The UI respected this rule — but one API endpoint did not.

Because of that mismatch, a regular user could retrieve project details by changing the `projectId`.

**Main lesson:** To reliably find IDORs, understand both user roles *and* feature states (like “shared but disabled by default”), and how these connect to backend authorization.


---

## The concept: “Shared” doesn’t always mean “accessible”

A subtle but common product pattern:

- The product supports Private Projects
- It also supports Shared Projects
- Whether Shared Projects are visible to everyone depends on workspace/org configuration
- In the default state, Shared Projects may exist as a type, but access is off unless an admin enables it

So the confusing-but-important rule becomes:

> “This project is marked as shared, but because the workspace default settings disable shared visibility, it must behave like private.”

That nuance is exactly where authorization bugs like IDORs appear.

---

## The fictional app: SprintForge

SprintForge is a project management platform with:

### Project types

- **Private Project:** only invited members can view
- **Shared Project:** broadly visible, but only if workspace settings allow it

### Workspace setting (default = OFF)

- **“Allow regular members to browse shared projects”:** Disabled by default  
  Meaning: regular users should not be able to view shared projects unless explicitly granted by role/config.

### Roles

- **Workspace Admin / Project Owner**
  - Can create projects and choose shared/private
- **Regular Member**
  - Can view only:
    - projects they are a member of
    - plus shared projects only if the workspace setting enables it

---

## Why this matters (impact)

Even “shared” projects can hold sensitive information, especially when sharing is controlled by configuration:

- internal roadmaps
- incident writeups
- customer escalations
- unreleased feature plans
- private client deliverables

If the backend ignores the configuration gate, it turns “shared-but-disabled” into “shared to anyone who knows the ID.”

---

## The core idea: map the role model and the default configuration

When I hunt IDORs, I explicitly write down:

1. Role model (admin vs regular vs guest)
2. Object model (workspace → projects → tickets/files)
3. Visibility model (private vs shared)
4. Configuration gates (shared visibility enabled/disabled by default)

Most testers stop at steps 1–3. In this case, the bug was hiding in step 4.

---

## What I observed: UI enforcement was correct

In SprintForge’s default configuration:

- A project owner created a Shared Project
- Workspace setting did not allow regular users to browse shared projects
- A regular user:
  - could not see the project in lists
  - could not open it via UI navigation
  - got “not authorized”/redirect when attempting to access the project screen

So the intended behavior was clear:

> The shared project exists, but for regular users it behaves like a private project unless the workspace enables shared browsing.

---

## The vulnerability: one API endpoint ignored the gate

While capturing traffic, I saw multiple endpoints for the same project object:

    GET /api/projects/{projectId}/activity   ✅ access checks enforced
    GET /api/projects/{projectId}/summary    ✅ access checks enforced
    GET /api/projects/{projectId}            ❌ returned details without checking the workspace gate

As a regular member (not a project member, and with shared browsing disabled), I requested:

    GET /api/projects/{projectId}

…and got back `200 OK` with project details.

### Why this is specifically “shared-but-should-be-private”

If the workspace had enabled shared browsing, access might be legitimate.

But with shared visibility disabled by default, authorization should have treated the project as private for non-members.

This is what made the issue tricky: it wasn’t “shared projects are accessible.” It was:

> Shared projects were accessible through the API even when workspace settings disabled shared visibility.

---

## Minimal reproduction steps (safe + controlled)

In an authorized test environment:

1. Admin/Owner: create a Shared Project in a workspace where shared browsing is disabled by default.
2. Regular user: confirm you cannot see/open the project in the UI.
3. Obtain the `projectId` (from owner-side traffic in the proxy or test fixtures).
4. As the regular user, send:

       GET /api/projects/{projectId}

5. If the API returns project metadata (title/description/status/etc.) instead of `403/404`, the IDOR is confirmed.

You don’t need brute force to prove this bug. A single unauthorized read is enough.

---

## How to spot this class of bug quickly

This isn’t just “swap IDs.” A reliable workflow:

### 1) Find a feature with a configuration gate

Examples:

- shared projects
- unlisted boards
- “company-wide visibility” toggles
- “view-only links”
- guest access modes

### 2) Verify the UI respects the gate

If the UI hides something, the backend should enforce the same rule.

### 3) Compare sibling endpoints

It’s common to see:

- `/summary` is protected
- `/details` is forgotten

Test endpoints returning the same underlying data:

    /api/projects/{id}
    /api/projects/{id}/settings
    /api/projects/{id}/members
    /api/projects/{id}/export
    /api/projects/{id}/files

### 4) Test both policy dimensions

You must check:

- membership (am I part of the project?)
- configuration (is shared browsing enabled?)

Many systems check one and forget the other.

---

## Likely root cause

A typical implementation mistake:

    if project.visibility == "shared":
        allow_access()

But it forgets the workspace gate:

    if project.visibility == "shared" and workspace.sharedBrowsingEnabled == true:
        allow_access()

Correct authorization must require either:

- user is a project member  
  OR
- project is shared AND shared browsing is enabled for that user’s role/scope

---

## Fix recommendations

### Deny-by-default

Authorization should default to deny unless all required policy conditions are met.

### Opaque IDs (hardening only)

Use UUIDs to reduce guesswork while still enforcing authorization.

### Regression tests for configuration gates

Add automated tests for:

- shared browsing disabled + shared project + non-member ⇒ 403/404
- shared browsing enabled + shared project + non-member ⇒ allowed (if policy says so)

---


## Takeaway

The best IDOR discoveries come from understanding how the app handles access, including defaults and configuration gates.

“Shared” usually isn’t a permission; it’s a state that still needs policy checks.
