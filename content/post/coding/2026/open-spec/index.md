---
title: "Building an app with OpenSpec"
date: 2026-05-22
tags: [ai, openspec]
description: ""
categories: [ code ]
draft: true
---


Building an app with **OpenSpec** (openspec.dev) involves a process called **Spec-Driven Development (SDD)**. Instead of jumping straight into coding, you define a "source of truth" in Markdown files that guide an AI agent (like Claude Code, Cursor, or GitHub Copilot) to build exactly what you need.

The core workflow follows a simple loop: **Propose → Apply → Archive**.

Note that you DON'T need to do full upfront specification of your application (waterfall) - you can just define and implement each feature one at a time.

---

### 1. Installation and Setup

First, you need to install the OpenSpec CLI globally and initialize it in your project folder.

* **Install:** Run `npm install -g @fission-ai/openspec@latest` in your terminal.
* **Initialize:** Navigate to your project directory and run `openspec init`.
* **Select your AI:** During setup, you will be prompted to choose which AI tool you are using (e.g., Claude Code, Cursor, or GitHub Copilot).

This creates an `openspec/` folder in your project with two main subdirectories:

* `specs/`: The "Source of Truth" representing how your system works right now.
* `changes/`: A workspace for new features or bug fixes currently in progress.

### 2. The Development Loop

#### Phase A: Propose (`/opsx:propose`)

When you want to build a new feature (e.g., a login page), you start by proposing it.

* **Command:** `/opsx:propose login-feature`
* **What happens:** OpenSpec creates a new folder in `openspec/changes/login-feature/` containing:
* `proposal.md`: Why this is being built and what the scope is.
* `design.md`: The technical approach (tech stack, architecture).
* `tasks.md`: A granular checklist of implementation steps.
* `specs/`: "Delta" specs showing exactly which requirements are being added or modified.



#### Phase B: Review and Refine

Before any code is written, you review the Markdown files. You can ask the AI to "update the design to use Tailwind CSS" or "add a task for unit testing." Changing Markdown is fast and free, whereas changing code later is expensive.

#### Phase C: Apply (`/opsx:apply`)

Once the plan looks solid, you tell the AI to execute.

* **Command:** `/opsx:apply`
* **What happens:** The AI agent reads `tasks.md` and begins writing the code step-by-step. It marks each task as completed as it goes. If a test fails, it stops and fixes the code based on the specification.

#### Phase D: Archive (`/opsx:archive`)

After the code is finished and verified, you merge the "proposed" changes into your "source of truth."

* **Command:** `/opsx:archive`
* **What happens:** OpenSpec moves the change files to an archive folder and updates the main `specs/` directory. Your documentation is now perfectly in sync with your new code.

### 3. Key Benefits for App Building

* **No "Vibe Coding":** The AI follows a strict contract (the spec) rather than guessing based on a vague prompt.
* **Persistence:** Unlike a chat history that might get lost, your specs live in your Git repo.
* **Parallel Work:** You can have different "changes" folders for different features, allowing you to work on multiple parts of an app without context collision.

[OpenSpec Changes Everything - No More Vibe Coding (Full Tutorial)](https://www.youtube.com/watch?v=5oUmpdpbejk)

This video provides a practical walk-through of integrating OpenSpec into a project and demonstrates how spec-driven development prevents the common pitfalls of AI-generated "vibe coding."


---

Adapting your requirements as you learn is a core strength of OpenSpec. Because it treats documentation as a "state machine," you aren't just editing text; you are evolving the system's contract.

When you realize a current specification is no longer accurate or needs to be refined for a new feature, you follow a structured **Change Request** workflow.

---

### 1. Identify the Target Spec

Before changing anything, identify which file in your `openspec/specs/` folder needs to be updated. This might be a `data-model.md`, a `user-flow.md`, or a `component-logic.md`.

### 2. Create a "Change" Proposal

Instead of editing the files in the `specs/` directory directly, you create a new change context. This prevents you from "breaking" the source of truth while you are still experimenting.

* **Command:** `/opsx:propose update-auth-logic`
* **The Delta:** In the newly created `openspec/changes/update-auth-logic/specs/` folder, you (or the AI) place a version of the spec file that reflects the *new* desired state.

### 3. The Reconciliation Process

Once the proposal is created, OpenSpec compares the "Current Spec" with your "Proposed Spec."

* **Conflict Detection:** If you try to change a requirement that contradicts another part of the system, the AI agent uses the `proposal.md` to reason through the conflict.
* **Refinement:** You can iterate on the `design.md` within that change folder. For example, if you learn that a specific API doesn't support the data format you initially planned, you update the design file first.

### 4. Executing the Adaptation (`/opsx:apply`)

When you run the apply command, the AI doesn't just look at your code; it looks at the **difference** between the old spec and the new spec.

* **Code Refactoring:** The AI identifies which files are affected by the requirement change.
* **Atomic Updates:** It begins refactoring the existing code to meet the new specifications. Because it has the `tasks.md` checklist, it ensures that "adapting" doesn't lead to "breaking" existing functionality.

### 5. Finalizing the Evolution (`/opsx:archive`)

The final step is the most important for maintaining a clean project. Running `/opsx:archive` performs a "merge."

* The modified spec files in your `changes/` folder move into the main `specs/` folder, overwriting the outdated requirements.
* The old version of the spec is moved to the `archive/` history.
* **Result:** Your `specs/` folder always represents the **current, learned reality** of the project, not the "day one" assumptions.

---

### Why this beats "Manual" Updating

In traditional development, documentation usually rots because developers update the code but forget to update the README or the Wiki. In OpenSpec, **the spec is the driver.** You cannot easily change the code without first defining the change in the spec, ensuring your "Source of Truth" evolves alongside your understanding of the problem.


----

Let’s walk through a practical scenario: building a **Leave Planner** dashboard. We will start with a basic requirement and then adapt it as we realize we need more functionality.

---

## Part 1: The Initial Spec

Imagine you want to build a simple view to see upcoming time off. You start the process by defining the "Source of Truth."

### 1. Propose the Feature

You run `/opsx:propose dashboard-view`. Inside `openspec/changes/dashboard-view/specs/`, you create `view-requirements.md`:

> **Feature:** Leave Dashboard
> **Goal:** Allow users to see a list of their upcoming leave.
> **Requirements:**
> * Display a list of leave entries.
> * Each entry must show: **Start Date**, **End Date**, and **Leave Type** (e.g., Annual, Sick).
> * Order the list chronologically by start date.
>
>

### 2. Design and Apply

You define the tech stack in `design.md` (e.g., utilizing Vue.js and Tailwind CSS). You then run `/opsx:apply`. The AI reads these requirements and generates the table component and the data fetching logic to match exactly what you wrote.

---

## Part 2: Modifying the Spec

After seeing the dashboard, you realize it’s missing a critical piece of information: the **Status** (Pending/Approved) of the request. You also want to add a visual indicator for different leave types.

### 1. Propose the Change

You don't just edit the code. You run `/opsx:propose add-leave-status`.

### 2. Update the Spec (The "Delta")

In the new change folder, you modify `view-requirements.md` to reflect the new reality:

> **Feature:** Leave Dashboard (Updated)
> **Requirements:**
> * Display a list of leave entries.
> * Each entry must show: Start Date, End Date, Leave Type, and **Approval Status**.
> * **Visual Polish:** Use color-coded badges for statuses (Green for Approved, Yellow for Pending).
> * **New Constraint:** Hide any leave entries that were "Rejected" by default.
>
>

### 3. The AI Refactor

Now, when you run `/opsx:apply`, the AI agent compares your *previous* spec with this *new* one. It identifies the gaps:

1. **Database/API:** It realizes it needs to fetch the `status` field.
2. **UI:** It adds the badge component to the existing table.
3. **Logic:** It adds a filter to the computed list to hide "Rejected" items.

### 4. Archive

Once you’re happy, you run `/opsx:archive`. The original `view-requirements.md` in your main `specs/` folder is updated to include the Status and Filtering requirements.

---

### Why this matters

By doing this, you've ensured that your documentation isn't just a "comment" on the code—it is the **blueprint** that generated it. If a new developer joins the project later, they don't have to hunt through the code to see why some items are hidden; they simply read the current Spec.

---

In the OpenSpec workflow, the file in your `changes/` directory should represent the **full, desired end-state** of that specific specification.

Think of it like a **Git patch** but for your requirements. When you are proposing a change, you aren't just sending "the new bits"—you are providing the updated version of the file as it should look once the work is finished.

---

### The "Full File" Strategy

When you run `/opsx:propose`, you (or the AI) should copy the existing spec from `openspec/specs/` into your `openspec/changes/[feature-name]/specs/` folder and then modify it.

* **Why?** It gives the AI agent the complete context. If you only provide a snippet of the change, the AI might lose track of existing constraints, logic, or dependencies that still need to be respected.
* **The Result:** After you run `/opsx:archive`, OpenSpec simply overwrites the old file in the main `specs/` folder with your new, complete version.

### Example Workflow

#### 1. The Original Spec (`specs/data-model.md`)

```markdown
# Data Model
- User: id, email, password_hash
- LeaveRequest: id, user_id, start_date, end_date

```

#### 2. The Learning Moment

You realize you need to track the "Status" of a leave request (Pending, Approved, Rejected).

#### 3. The Proposed Change (`changes/add-status/specs/data-model.md`)

You don't just write "Add status to LeaveRequest." You write the **entire updated model**:

```markdown
# Data Model
- User: id, email, password_hash
- LeaveRequest: id, user_id, start_date, end_date, status (enum)

```

### Why this matters for AI Agents

When you use a tool like Cursor or Claude Code with OpenSpec:

1. **State Comparison:** The agent looks at the "Main Spec" (what is currently true) and the "Proposed Spec" (what you want).
2. **Implementation:** It sees the delta (the addition of `status`) and knows exactly what to change in your Kotlin files, database schema, or Vue components.
3. **Validation:** It ensures that adding `status` doesn't accidentally break the existing `start_date` or `end_date` logic because those are still present in the file.

### Summary of the "Mental Model"

* **`openspec/specs/`**: This is your "Current Production" documentation.
* **`openspec/changes/`**: This is your "Drafting Table." The files here are the final versions of what you want "Production" to look like after the task is done.

By keeping the specs complete, you ensure the AI never "forgets" the original requirements while it’s busy building the new ones.



