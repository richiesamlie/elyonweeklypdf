# Deployment Checklist (Lightweight)

Use this quick smoke-check after every Google Apps Script deployment.

## 1) Deployment Access

- Execute as: `User accessing the web app`
- Who has access: `Only @elyon.sch.id`

## 2) App Startup

- Open web app URL.
- Verify grade chips render.
- Verify week dropdown loads for selected grade.

## 3) Export Smoke Test

- Select 1 class and export PDF.
- Select multiple classes and export ZIP.
- Confirm downloaded file(s) open normally.

## 4) Editor Smoke Test

- Switch to Program Editor.
- Open one class/week.
- Change one materials or note field.
- Save and verify success status.
- Verify last-edited stamp updates.

## 5) Week Tab Operations

- Create a test week tab using New Week Tab.
- Confirm the tab appears.
- Delete the same test tab.
- Confirm deletion succeeds.

## 6) Cache Refresh

- Click `Refresh` and verify level data reloads.
- Confirm updated cache status timestamp is shown.

## 7) Failure Snapshot (If Any)

If create/delete tab fails, capture:

- Grade level
- Week name
- List of failed class names from error message
- Time of failure

This data is enough for quick retry and diagnosis.
