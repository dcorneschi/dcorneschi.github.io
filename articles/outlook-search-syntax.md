# Outlook Instant Search Syntax

A reference guide for Outlook's Instant Search query syntax — operators, field keywords, date ranges, attachment filters, and contact/calendar searches.

## General Rules

- Instant Search is **not case sensitive**
- Logical operators (`AND`, `NOT`, `OR`) **must be in uppercase**
- Use double quotes for exact phrase matching
- Use parentheses to group conditions

## Basic Search

| Type this | To find this |
|-----------|-------------|
| `bob` | Items containing bob, bobbin, bobby, BOBBY, or any case combination |
| `bob moore` | Items containing bob or moore (not necessarily in that order) |
| `bobby AND moore` | Items containing both bobby and moore |
| `bobby NOT moore` | Items containing bobby but not moore |
| `bobby OR moore` | Items containing bobby, moore, or both |
| `"bob"` | Items containing the exact phrase bob (not bobby or bobbin) |

## From / To / Cc / Bcc

| Type this | To find this |
|-----------|-------------|
| `from:"bobby moore"` | Items sent from bobby moore |
| `from:"bobby moore" about:"status report"` | Items from bobby moore where "status report" appears in subject, body, or attachments |
| `to:bobby` | Items sent to bobby (search in Sent Items folder) |
| `cc:"bobby moore"` | Items with bobby moore on the Cc line |
| `cc:bobbymoore@contoso.com` | Items with that email on the Cc line |
| `bcc:bobby` | Items with bobby on the Bcc line |

## Subject

| Type this | To find this |
|-----------|-------------|
| `subject:"bobby moore"` | Items whose subject contains the phrase "bobby moore" |
| `subject:status received:May` | Items received in May (any year) with "status" in the subject |

## Attachments

| Type this | To find this |
|-----------|-------------|
| `hasattachment:yes` | Items that have attachments (also: `hasattachment:true`) |
| `attachments:presentation.pptx` | Items with that attachment name or containing it in attachment contents |
| `ext:docx` | Items with .docx attachments |
| `ext:pdf` | Items with .pdf attachments |
| `ext:xlsx` | Items with .xlsx attachments |
| `ext:pptx` | Items with .pptx attachments |
| `ext:zip` | Items with .zip attachments |
| `ext:(docx pdf)` | Items with .docx or .pdf attachments |
| `ext:(doc NOT docx)` | Items with .doc but not .docx attachments |
| `attachment:outlook` | Items with attachments containing "outlook" |
| `attachment:(my search words)` | Items with attachments matching the search words |

## Message Size

| Type this | To find this |
|-----------|-------------|
| `messagesize:<10 KB` | Items smaller than 10 KB |
| `messagesize:>5 MB` | Items larger than 5 MB |
| `messagesize:tiny` | Less than 10 KB |
| `messagesize:small` | Between 10 and 25 KB |
| `messagesize:medium` | Between 25 and 100 KB |
| `messagesize:large` | Between 100 and 500 KB |
| `messagesize:verylarge` | Between 500 KB and 1 MB |
| `messagesize:enormous` | Larger than 5 MB |

## Dates (Received / Sent)

| Type this | To find this |
|-----------|-------------|
| `received:=1/1/2016` | Items that arrived on 1/1/2016 |
| `received:yesterday` | Items that arrived yesterday |
| `received:last week` | Items that arrived last week |
| `sent:yesterday` | All items sent yesterday |

### Supported Date Values

- **Relative dates:** today, tomorrow, yesterday
- **Multi-word relative dates:** this week, next month, last week, past month, coming year
- **Days:** Sunday, Monday, Tuesday, Wednesday, Thursday, Friday, Saturday
- **Months:** January, February, March, April, May, June, July, August, September, October, November, December

### Date Ranges

| Type this | To find this |
|-----------|-------------|
| `received>=10/1/16 AND received<=10/5/16` | Items arrived between 10/1/16 and 10/5/16 |
| `received>10/1/16 AND received<10/5/16` | Items arrived after 10/1/16 but before 10/5/16 |
| `from:bobby (received:1/7/17 OR received:1/8/17)` | Items from bobby on either date |

> **Note:** For received ranges, do not use a colon. Use comparison operators directly (e.g., `received>=`).

## Flags and Read Status

| Type this | To find this |
|-----------|-------------|
| `hasflag:true` | Items flagged for follow up |
| `followupflag:follow up` | Items flagged with the Follow Up flag |
| `due:last week` | Items flagged with a due date last week |
| `read:no` | Unread items (also: `read:false`) |

## Categories

| Type this | To find this |
|-----------|-------------|
| `category:red` | Items with a category name containing "red" (e.g., "Red category", "Redo", "Redundant") |

## Calendar Searches

These searches only return proper results when run from a **Calendar folder**.

| Type this | To find this |
|-----------|-------------|
| `startdate:next week subject:status` | Calendar items next week with "status" in the subject |
| `is:recurring` | Recurring calendar items |
| `organizer:bobby` | Calendar items where bobby is the organizer |

## Contact Searches

These searches only return proper results when run from a **Contacts folder**.

| Type this | To find this |
|-----------|-------------|
| `firstname:bobby` | Contacts with "bobby" in the First Name field |
| `lastname:moore` | Contacts with "moore" in the Last Name field |
| `nickname:bobby` | Contacts with "bobby" in the Nickname field |
| `jobtitle:physician` | Contacts with "physician" in the Job Title field |
| `businessphone:555-0100` | Contacts with "555-0100" in the Business Phone field |
| `homephone:555-0100` | Contacts with "555-0100" in the Home Phone field |
| `mobilephone:555-0100` | Contacts with "555-0100" in the Mobile Phone field |
| `businessfax:555-0100` | Contacts with "555-0100" in the Business Fax field |
| `businessaddress:(4567 Main St., Buffalo, NY 98052)` | Contacts matching that business address |
| `homeaddress:(4567 Main St., Buffalo, NY 98052)` | Contacts matching that home address |
| `businesscity:buffalo` | Contacts with "buffalo" in the Business City field |
| `businesspostalcode:98052` | Contacts with "98052" in the Business Postal Code field |
| `street:(4567 Main St)` | Contacts matching that business street address |
| `homestreet:(4567 Main St)` | Contacts matching that home street address |
| `birthday:6/4/1960` | Contacts with that birthday |
| `webpage:www.contoso.com` | Contacts with that URL in the Web Page Address field |
