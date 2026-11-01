# n8n Student Fee Management -- AI-Powered CSV ETL Automation

## Assignment Overview

**Project:** Student Enrollment CSV Cleaning and Fee Status Segregation
using n8n and AI\
**Platform:** n8n (self-hosted)\
**Input:** CSV file uploaded through an n8n Form Trigger\
**Processing:** Switch, LLM, Loop Over Items, JavaScript Code, IF\
**Output:** Clean student CSV files separated by fee-payment status

## 1. Objective

Build an n8n workflow that accepts a student-enrollment CSV file, checks
whether a file was uploaded, uses an LLM to identify and clean messy
student records, validates and refines each record with JavaScript, and
separates students into paid-fee and pending-fee CSV files.

The workflow demonstrates an ETL process:

-   **Extract:** Read the uploaded CSV file and its student records.
-   **Transform:** Use an LLM and JavaScript to clean, standardize,
    validate, and refine the data.
-   **Load:** Generate separate CSV output files for students who have
    paid their fees and students whose fees are pending.

## 2. Workflow Architecture

``` text
Form Trigger – CSV Upload
          |
          v
Switch – Was a file uploaded?
       /       \
    True       False
     |           |
     v           v
Extract CSV   Return upload
(file data)   validation message
     |
     v
Parse CSV into student records
     |
     v
LLM – Clean and standardize records
     |
     v
Loop Over Items – Process each student
     |
     v
JavaScript – Validate and refine fields
     |
     v
IF – Is fee status PAID?
       /       \
     True      False
      |          |
      v          v
  Paid group   Pending group
      |          |
      v          v
Convert to CSV Convert to CSV
      |          |
      v          v
Paid students  Pending students
CSV file       CSV file
```

## 3. Workflow Steps

### Step 1 -- Upload the CSV file

Use an n8n Form Trigger with a File Upload field.

Example input filename:

`student_enrolment_raw.csv`

The CSV should contain at least these columns:

  Student_ID   Name      Email               Course         Fee_Status
  ------------ --------- ------------------- -------------- ------------
  STU1001      Manoj K   manoj@example.com   Python         paid
  STU1002      Priya S   PRIYA@EXAMPLE.COM   Data Science   pending
  STU1003      Arjun R   arjun@example.com   Python         PAID

The source data may intentionally contain inconsistent capitalization,
extra spaces, misspellings, blank values, or inconsistent fee-status
labels.

### Step 2 -- Switch: Check whether a file was uploaded

Add a Switch node after the Form Trigger. The purpose is to route the
workflow based on whether the File Upload field contains an uploaded
file.

For a Form Trigger field that returns an array of uploaded files, use
this expression as the value to evaluate:

``` javascript
{{ Array.isArray($json["File Upload"]) ? $json["File Upload"].length : ($json["File Upload"] ? 1 : 0) }}
```

Configure the rules using **Number** comparisons:

-   **File uploaded route:** is greater than `0`
-   **No file route:** is equal to `0`

Do not use `is empty` or `is not empty` for this numeric expression. If
the form field has a different name or its output structure differs,
select the actual field from the Form Trigger's input panel and adapt
the expression.

The no-file route should return a clear validation message such as:

`Please upload a valid student CSV file before continuing.`

**Important:** In the Form Trigger output, inspect whether `File Upload`
is an array of file objects and identify the binary property used by the
next node. The Switch only checks file presence; it does not extract the
CSV contents.

### Step 3 -- Extract and parse the CSV

On the true route:

1.  Access the uploaded file's binary data.
2.  Use the appropriate n8n file extraction / CSV parsing node to read
    the CSV.
3.  Convert each CSV row into an individual n8n item.
4.  Confirm that the parsed output contains the expected student fields.

The exact binary property name depends on the Form Trigger output. Use
the binary field shown in the execution data rather than assuming its
name.

### Step 4 -- LLM: Clean and standardize student records

Send the parsed student data to an LLM node. Ask the model to clean
values while preserving the original student information.

Suggested LLM instruction:

``` text
You are a student enrollment data-cleaning assistant.

Clean the supplied student records and return valid structured JSON.

Requirements:
1. Trim leading and trailing spaces from every text field.
2. Standardize student IDs to uppercase.
3. Standardize email addresses to lowercase.
4. Correct obvious capitalization issues in student names and course names.
5. Normalize fee status to exactly PAID or PENDING.
6. Map clear variations such as "paid", "PAID ", "completed" to PAID,
   and "pending", "not paid", "unpaid" to PENDING.
7. Do not invent missing student IDs, names, email addresses, courses,
   or payment information.
8. If a fee status is unclear or missing, mark it as REVIEW.
9. Preserve one output record for each input student.
10. Return JSON only, using this structure:

{
  "students": [
    {
      "Student_ID": "STU1001",
      "Name": "Manoj K",
      "Email": "manoj@example.com",
      "Course": "Python",
      "Fee_Status": "PAID"
    }
  ]
}

Do not claim that a fee was paid unless the source data supports it.
```

For reliable processing, configure a Structured Output Parser or
otherwise ensure the LLM returns machine-readable JSON. Do not pass
explanatory prose or Markdown fences into the next data-processing step.

### Step 5 -- Loop Over Items: Process each student

Use a Loop Over Items node to process each student record individually.

Inside the loop:

-   Send one student record at a time to the JavaScript Code node.
-   Validate mandatory fields.
-   Normalize text values.
-   Check email format.
-   Normalize fee status.
-   Send the refined student record to the next node.

Connect the processing path back to the Loop Over Items node as required
by the n8n loop pattern. Use the loop's **Done** output when all records
have been processed.

### Step 6 -- JavaScript: Validate and refine each record

Example Code node logic for one student item:

``` javascript
const student = $json;

const clean = (value) =>
  String(value ?? "").trim();

const status = clean(student.Fee_Status).toUpperCase();

const email = clean(student.Email).toLowerCase();

const validEmail = /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);

const refined = {
  Student_ID: clean(student.Student_ID).toUpperCase(),
  Name: clean(student.Name),
  Email: email,
  Course: clean(student.Course),
  Fee_Status: ["PAID", "PENDING"].includes(status)
    ? status
    : "REVIEW",
  Validation_Status: "VALID"
};

if (
  !refined.Student_ID ||
  !refined.Name ||
  !refined.Course ||
  !validEmail
) {
  refined.Validation_Status = "REVIEW";
}

return {
  json: refined
};
```

This code validates the record; it does not independently verify a bank
transaction or payment receipt. Fee status must be based on trusted
source data.

### Step 7 -- IF: Check whether the student has paid fees

Add an IF node after the record-processing stage.

Configure:

-   **Value 1:** `{{ $json.Fee_Status }}`
-   **Operation:** String → is equal to
-   **Value 2:** `PAID`

Route the true output to the paid-students group. Route the false output
to the pending/review handling path.

For a safer workflow, use a second IF or Switch to separate `PENDING`
from `REVIEW`, rather than treating every non-PAID record as unpaid.

### Step 8 -- Generate separate CSV files

Create separate output datasets:

**Paid students**

-   Student_ID
-   Name
-   Email
-   Course
-   Fee_Status

Filename:

`students_paid_fees.csv`

**Pending students**

-   Student_ID
-   Name
-   Email
-   Course
-   Fee_Status

Filename:

`students_pending_fees.csv`

Use the appropriate n8n aggregation / data-collection approach to gather
all records in each branch, then use **Convert to File** with CSV format
to generate each file. Ensure the workflow handles an empty branch
gracefully.

Optionally, generate a third file for records needing manual review:

`students_review_required.csv`

## 4. Expected Results

After a successful execution, the workflow should produce:

  -----------------------------------------------------------------------
  Output file                         Records included
  ----------------------------------- -----------------------------------
  `students_paid_fees.csv`            Students whose normalized fee
                                      status is PAID

  `students_pending_fees.csv`         Students whose normalized fee
                                      status is PENDING

  `students_review_required.csv`      Records with missing, invalid, or
                                      unclear data (optional)
  -----------------------------------------------------------------------

The number of records across all output files should equal the number of
input student records, unless a documented duplicate-removal rule is
deliberately added.

## 5. Testing and Validation

Test the workflow with at least these scenarios:

1.  A valid CSV file containing both PAID and PENDING students.
2.  No file uploaded.
3.  A CSV with extra spaces and inconsistent capitalization.
4.  A CSV with uppercase and mixed-case email addresses.
5.  A student record with a missing email address.
6.  A student record with an unclear or missing fee status.
7.  A CSV where all students have paid.
8.  A CSV where no students have paid.

Verify that:

-   The Switch sends uploaded files through the true route.
-   The CSV parser extracts the expected rows.
-   The LLM returns valid structured data.
-   The loop processes every student record.
-   JavaScript flags invalid records for review.
-   The IF node routes PAID records correctly.
-   The generated CSV files contain the correct records.

## 6. Error Handling and Improvements

-   Validate the uploaded file type and reject files that are not CSV.
-   Handle empty files and malformed CSV rows.
-   Use structured LLM output to reduce JSON parsing failures.
-   Keep the original input data available for audit and
    troubleshooting.
-   Do not allow the LLM to invent missing student or payment details.
-   Route REVIEW records for manual verification.
-   Avoid sending actual student personal information to an external LLM
    unless the organization has approved the data-sharing arrangement.
-   Log execution errors and record counts without unnecessarily
    exposing student personal data.

## 7. Conclusion

This assignment implements an AI-powered ETL automation workflow in n8n.
It extracts student records from an uploaded CSV, uses an LLM to clean
and standardize messy data, validates each record with JavaScript, and
separates paid and pending fee records into individual CSV files.

The solution demonstrates practical use of workflow automation,
AI-assisted data cleaning, conditional routing, record-by-record
processing, validation, and automated file generation.
