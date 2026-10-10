# Personal Dashboard

A single-user dashboard that pulls together data about one person's life from their own sources.

## Language

**Dashboard**:
The whole application, organised into tabs. Single-user: it belongs to Ganesh.
_Avoid_: app, portal

**Life in Weeks Calendar**:
A grid in which each cell is one **Week** of Ganesh's life, from birth to age 97, used to show what happened when.
_Avoid_: life calendar, timeline

**Week**:
One cell of the **Life in Weeks Calendar**.

**Journal Entry**:
One daily page in Ganesh's Logseq journal, written in free text.
_Avoid_: note, diary page

**Reading**:
The period over which one book was read, from the week it was started to the week it was finished.
_Avoid_: book (the book is the thing being read; the **Reading** is the time spent reading it)

**AG**:
Ambika and Ganesh as a household, as in "AG Hub", the household's shared documents and admin rules.

**Auto**:
Ganesh's car: its costs and maintenance cycles.
_Avoid_: automations

## Relationships

- A **Reading** spans one or more **Weeks** on the **Life in Weeks Calendar**
- **Journal Entries** are evidence of when a **Reading** started or finished

## Flagged ambiguities

- "AG Hub" is a household concept, but the **Dashboard** is single-user. This is unresolved because AG Hub is out of scope for now.
