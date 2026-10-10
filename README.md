# Student Skill Development Tracker (Zoho Creator)

A low-code application built on **Zoho Creator** to register students, enrol them in skill-training programs, assess their progress, and use **AI fields** to extract insight from certificates, feedback and self-descriptions.

---

## 1. Project Description

Colleges run many skill-development activities (coding bootcamps, communication workshops, aptitude training, certifications), but records are usually scattered across spreadsheets and emails. This application centralises everything in one place:

- Students register and build a skill profile.
- Coordinators create training programs with seat limits.
- Students enrol; assessments are recorded and levels are calculated automatically.
- Certificates are uploaded and read by AI (OCR), and feedback is analysed by AI (sentiment and keywords).
- Automated emails, approvals and reminders keep everyone informed.

**Platform:** Zoho Creator (forms, reports, workflows, Deluge scripting, AI fields)

---

## 2. Problem Statements

| # | Problem | How the app solves it |
|---|---------|----------------------|
| 1 | Student skill data is scattered across spreadsheets, so there is no single view of a student's progress. | A central **Student** and **Skill Profile** record per student. |
| 2 | Manual enrolment causes duplicates and over-booking of limited seats. | Validation scripts block duplicate students and full programs. |
| 3 | Assessment results are graded by hand and levels are inconsistent. | Deluge calculates percentage and assigns Beginner / Intermediate / Advanced. |
| 4 | Coordinators must read every certificate and feedback entry manually. | **AI fields** read certificate text (OCR) and score feedback sentiment automatically. |
| 5 | Students forget pending assessments; negative feedback goes unnoticed. | Scheduled reminders and instant alerts to coordinators. |
| 6 | Certificate verification is slow and untracked. | Approval workflow with status tracking. |

---

## 3. Forms

### 3.1 Basic Fields (standard form fields)

#### Form: `Student`
| Field | Type | Notes |
|-------|------|-------|
| Full_Name | Single Line | Mandatory |
| Register_No | Single Line | Mandatory, unique |
| Email | Email | Mandatory, unique |
| Phone | Phone Number | |
| Department | Dropdown | CSE, AIML, ECE, MECH, etc. |
| Year | Radio | 1 / 2 / 3 / 4 |
| Self_Description | Multi Line | Source for AI keyword extraction |

#### Form: `Training_Program`
| Field | Type | Notes |
|-------|------|-------|
| Program_Name | Single Line | |
| Skill_Category | Dropdown | Technical, Soft Skills, Aptitude, Domain |
| Trainer | Single Line | |
| Start_Date / End_Date | Date | End date must be after start date |
| Seat_Limit | Number | |
| Status | Dropdown | Upcoming, Ongoing, Completed |

#### Form: `Enrollment`
| Field | Type | Notes |
|-------|------|-------|
| Student | Lookup (Student) | |
| Program | Lookup (Training_Program) | |
| Enrolled_On | Date | Default: today |
| Status | Dropdown | Enrolled, In Progress, Completed, Dropped |

#### Form: `Assessment`
| Field | Type | Notes |
|-------|------|-------|
| Student | Lookup (Student) | |
| Program | Lookup (Training_Program) | |
| Score | Decimal | |
| Max_Score | Decimal | |
| Percentage | Decimal | Filled by script |
| Level | Dropdown | Filled by script |
| Assessed_On | Date | |

#### Form: `Skill_Profile`
| Field | Type | Notes |
|-------|------|-------|
| Student | Lookup (Student) | One profile per student |
| Overall_Level | Dropdown | Beginner, Intermediate, Advanced |
| Programs_Completed | Number | Updated by script |

#### Form: `Certificate`
| Field | Type | Notes |
|-------|------|-------|
| Student | Lookup (Student) | |
| Program | Lookup (Training_Program) | |
| Certificate_File | Image / File Upload | Input for OCR AI field |
| Approval_Status | Dropdown | Pending, Approved, Rejected |
| Remarks | Multi Line | |

#### Form: `Feedback`
| Field | Type | Notes |
|-------|------|-------|
| Student | Lookup (Student) | |
| Program | Lookup (Training_Program) | |
| Rating | Rating / Number | 1 to 5 |
| Comments | Multi Line | Input for AI fields |

### 3.2 AI Fields

Zoho Creator AI fields process another field's content automatically. Availability depends on your Creator plan, so confirm in the form builder under the AI field group.

| Form | AI Field | AI Type | Input Field | Purpose |
|------|----------|---------|-------------|---------|
| Certificate | `Certificate_Text` | Text Recognition (OCR) | Certificate_File | Extracts the text printed on the certificate so coordinators can verify the name, course and date without opening the image. |
| Feedback | `Feedback_Sentiment` | Sentiment Analysis | Comments | Classifies feedback as Positive, Neutral or Negative. |
| Feedback | `Feedback_Keywords` | Keyword Extraction | Comments | Surfaces recurring topics (for example "pace", "trainer", "labs"). |
| Student | `Skill_Keywords` | Keyword Extraction | Self_Description | Pulls out the skills a student claims, for matching to programs. |

---

## 4. Deluge Scripts

> Field link names above are assumed. Adjust them to match your application's link names.

### 4.1 Student: On Validate (block duplicates)
```deluge
if(count(Student[Email == input.Email]) > 0)
{
    alert "A student with this email already exists.";
    cancel submit;
}
if(count(Student[Register_No == input.Register_No]) > 0)
{
    alert "This register number is already registered.";
    cancel submit;
}
```

### 4.2 Student: On Success (create profile and welcome mail)
```deluge
insert into Skill_Profile
[
    Student = input.ID
    Overall_Level = "Beginner"
    Programs_Completed = 0
    Added_User = zoho.loginuser
];

sendmail
[
    from : zoho.adminuserid
    to : input.Email
    subject : "Welcome to the Skill Development Program"
    message : "Hi " + input.Full_Name + ",<br>Your skill profile has been created. You can now enrol in training programs."
];
```

### 4.3 Enrollment: On Validate (seat limit and duplicate enrolment)
```deluge
program = Training_Program[ID == input.Program];
enrolled = Enrollment[Program == input.Program && Status != "Dropped"].count();

if(enrolled >= program.Seat_Limit)
{
    alert "Seats are full for " + program.Program_Name + ".";
    cancel submit;
}
if(count(Enrollment[Student == input.Student && Program == input.Program]) > 0)
{
    alert "This student is already enrolled in this program.";
    cancel submit;
}
```

### 4.4 Assessment: On Validate (score check)
```deluge
if(input.Score > input.Max_Score || input.Score < 0)
{
    alert "Score must be between 0 and the maximum score.";
    cancel submit;
}
```

### 4.5 Assessment: On Success (grade and update profile)
```deluge
pct = (input.Score / input.Max_Score) * 100;

if(pct >= 80)
{
    lvl = "Advanced";
}
else if(pct >= 50)
{
    lvl = "Intermediate";
}
else
{
    lvl = "Beginner";
}

input.Percentage = pct.round(2);
input.Level = lvl;

profile = Skill_Profile[Student == input.Student];
if(profile.count() > 0)
{
    profile.Overall_Level = lvl;
}

// Mark the enrolment as completed
enr = Enrollment[Student == input.Student && Program == input.Program];
if(enr.count() > 0)
{
    enr.Status = "Completed";
}
```

### 4.6 Feedback: On Success (alert on negative sentiment)
```deluge
// AI fields can take a moment to populate, so re-fetch the record
fb = Feedback[ID == input.ID];

if(fb.Feedback_Sentiment == "Negative" || fb.Rating <= 2)
{
    sendmail
    [
        from : zoho.adminuserid
        to : "coordinator@yourcollege.edu"
        subject : "Negative feedback received: " + fb.Program.Program_Name
        message : "A student submitted low-rated feedback.<br>Comments: " + fb.Comments
    ];
}
```

### 4.7 Certificate: On Approval Change (notify student)
```deluge
if(input.Approval_Status == "Approved")
{
    sendmail
    [
        from : zoho.adminuserid
        to : input.Student.Email
        subject : "Your certificate has been approved"
        message : "Hi " + input.Student.Full_Name + ", your certificate for " + input.Program.Program_Name + " is approved."
    ];

    profile = Skill_Profile[Student == input.Student];
    if(profile.count() > 0)
    {
        profile.Programs_Completed = profile.Programs_Completed + 1;
    }
}
```

### 4.8 Schedule: Weekly Reminder for Pending Assessments
```deluge
pending = Enrollment[Status == "Enrolled" || Status == "In Progress"];

for each e in pending
{
    sendmail
    [
        from : zoho.adminuserid
        to : e.Student.Email
        subject : "Reminder: pending assessment"
        message : "Hi " + e.Student.Full_Name + ", your assessment for " + e.Program.Program_Name + " is still pending."
    ];
}
```

---

## 5. Workflows

| # | Workflow | Form | Trigger | What happens |
|---|----------|------|---------|--------------|
| 1 | Student Registration | Student | On Add, validate, then success | Duplicate check, creates Skill Profile, sends welcome email (4.1, 4.2). |
| 2 | Program Enrolment | Enrollment | On Add, validate | Blocks full programs and duplicate enrolment (4.3). |
| 3 | Assessment Grading | Assessment | On Add, validate, then success | Validates score, calculates percentage and level, updates profile and enrolment status (4.4, 4.5). |
| 4 | Certificate Approval | Certificate | On Add, then Approval | Certificate enters **Pending**, OCR AI field reads the text, coordinator approves or rejects, student is emailed (4.7). |
| 5 | Feedback Monitoring | Feedback | On Add, success | AI sentiment and keywords are generated, and negative feedback alerts the coordinator (4.6). |
| 6 | Weekly Reminder | Schedule | Every Monday, 9:00 AM | Emails students with pending assessments (4.8). |
| 7 | Program Status Update | Training_Program | Daily schedule | Sets status to Ongoing or Completed based on start and end dates. |

### Workflow Flow

```
Student registers ──► Skill Profile created ──► Enrols in Program
                                                      │
                                   Assessment recorded ──► Level auto-calculated
                                                      │
                          Certificate uploaded ──► AI (OCR) ──► Coordinator approval
                                                      │
                               Feedback submitted ──► AI sentiment / keywords ──► Alerts
```

---

## 6. Reports and Dashboard (suggested)

- Students by department and year
- Program-wise enrolment vs seat limit
- Level distribution (Beginner / Intermediate / Advanced)
- Certificate approval status
- Feedback sentiment breakdown and top keywords

---

## 7. How to Set Up

1. Create a new Zoho Creator application named **Student Skill Development**.
2. Build the forms in Section 3 with the listed field link names.
3. Add the AI fields to Certificate, Feedback and Student.
4. Paste each Deluge script into the matching form workflow (Section 4).
5. Configure the two schedules (weekly reminder and daily program status).
6. Create the reports and dashboard in Section 6.
7. Share the app with roles: **Admin**, **Coordinator**, **Student**.

---

## 8. Tech Stack

- Zoho Creator (low-code platform)
- Deluge (scripting)
- Zoho Creator AI fields (OCR, sentiment analysis, keyword extraction)
