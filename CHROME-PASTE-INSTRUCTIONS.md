# Chrome Claude Instructions: Replace Semester 2 Sections

These instructions are for Claude in Chrome to execute on the Google Doc version of Atwani's final report.

**Source file for all new content:** `~/Desktop/CSIT321/term2_files/RealSync/FINAL-SEMESTER2-SECTIONS.md`

**Google Doc:** The finalized report (Atwani's version) in Google Drive.

---

## Important rules

- Do NOT touch anything in the semester 1 sections (Planning & Feasibility, SRS, Software Design Document)
- Only replace/add in the semester 2 sections (Change Report, Test Plan, Test Results, Implementation)
- Tables must be pasted as formatted tables in Google Docs, not raw markdown
- Preserve heading hierarchy (H1 for document titles, H2 for numbered sections, H3 for subsections)

---

## Step-by-step operations

### Operation 1: Replace the Change Report

1. **Find (CMD+F):** `Change Report`
2. **Locate the section** starting with heading "Change Report" (after the Test Plan Document section)
3. **Select everything** from the "Change Report" heading down to (but NOT including) the "Test Result Report" heading
4. **Delete** the selected content
5. **Paste** the "CHANGE REPORT" section from `FINAL-SEMESTER2-SECTIONS.md`
   - This starts at `# Change Report` and ends before `# Test Plan Document`

### Operation 2: Replace the Test Plan Document

1. **Find (CMD+F):** `Test Plan Document`
2. **Locate the section** starting with heading "Test Plan Document"
3. **Select everything** from "Test Plan Document" heading down to (but NOT including) the "Change Report" heading
4. **Delete** the selected content
5. **Paste** the "TEST PLAN DOCUMENT" section from `FINAL-SEMESTER2-SECTIONS.md`
   - This starts at `# Test Plan Document` and ends before `# Change Report`

### Operation 3: Replace the Test Result Report

1. **Find (CMD+F):** `Test Result Report`
2. **Locate the section** starting with heading "Test Result Report"
3. **Select everything** from "Test Result Report" heading down to (but NOT including) "Final Presentation Slides" heading
4. **Delete** the selected content
5. **Paste** the "TEST RESULT REPORT" section from `FINAL-SEMESTER2-SECTIONS.md`
   - This starts at `# Test Result Report` and ends before `# Implementation Report`

### Operation 4: Add the Implementation Report (NEW - Atwani doesn't have this)

1. **Find (CMD+F):** `Test Result Report`
2. **Go to the END** of the Test Result Report section (after the Conclusion subsection)
3. **Position cursor** AFTER the Test Result Report's conclusion, BEFORE "Final Presentation Slides"
4. **Paste** the "IMPLEMENTATION REPORT" section from `FINAL-SEMESTER2-SECTIONS.md`
   - This starts at `# Implementation Report` and ends before the file ends

### Operation 5: Verify section order

After all operations, the semester 2 sections should appear in this order:

1. *(semester 1 sections - untouched)*
2. Test Plan Document
3. Change Report
4. Test Result Report
5. **Implementation Report** ← NEW
6. Final Presentation Slides *(placeholder - to be filled separately)*
7. Marketing Poster *(placeholder - to be filled separately)*
8. References

### Operation 6: Update Table of Contents

1. **Find (CMD+F):** `Table of Contents`
2. Add an entry for "Implementation Report" after "Test Result Report" in the TOC
3. Verify page numbers will update when printed (Google Docs auto-updates linked TOCs)

---

## Verification checklist

After pasting, verify:

- [ ] Change Report has the architecture changes table with 11 rows (Video detection through Ensemble detection)
- [ ] Change Report has the requirements changes table with FR1-FR10
- [ ] Change Report has the cost changes table
- [ ] Change Report has sections 8.4 (Why changes were made) and 8.5 (Puppeteer to Recall.ai migration)
- [ ] Test Plan has FR1-FR10 test case tables with EMPTY Test Results and Verdict columns
- [ ] Test Plan has the End-to-End Integration Testing section (section 7)
- [ ] Test Result Report has FR1-FR10 tables with FILLED Test Results and Verdict columns
- [ ] Test Result Report has the End-to-End Integration Test Results section with the 4-scenario table
- [ ] Test Result Report has the 360p Calibration Results subsection
- [ ] Implementation Report has sections: AI detection pipeline, Zoom bot, Backend, Frontend
- [ ] No semester 1 content was modified
- [ ] No duplicate sections exist

---

## Screenshot placeholders

The following placeholders appear in the text as `[SCREENSHOT: description]`. Ahmed needs to take these screenshots and insert them:

1. `[SCREENSHOT: Dashboard showing real face with LOW risk]` — in FR3-01 or E2E Test 1
2. `[SCREENSHOT: Dashboard showing deepfake with HIGH risk and alerts]` — in FR3-02 or E2E Test 2
3. `[SCREENSHOT: Deep Live Cam + OBS Virtual Camera setup]` — in E2E test setup
4. `[SCREENSHOT: RealSync Bot appearing in Zoom participant list]` — in FR2-01
5. `[SCREENSHOT: Login page]` — in FR1-01
6. `[SCREENSHOT: Sessions page with session list]` — in FR2
7. `[SCREENSHOT: Reports page with generated PDF]` — in FR7-01
8. `[SCREENSHOT: Settings page with notification toggles]` — in FR9
9. `[SCREENSHOT: OS notification alert banner]` — in FR9-01
10. `[SCREENSHOT: Supabase database tables]` — in Implementation or FR5
11. `[SCREENSHOT: RunPod GPU dashboard]` — in E2E test environment
