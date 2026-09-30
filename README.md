# NETWORKWALKS-SEMILORE-B083-WK4-PM1-MEDIROZA-PENETRATION-TEST

# Penetration Testing Project: Mediroza General Hospital

**Pentester:** Aboderin Semilore Gold
**Program:** Networkwalks, Cybersecurity (Batch B083)
**Date:** 30 September 2026
**Target:** https://medirozahospital.com
**Engagement Type:** Black-box Penetration Test
**Duration:** 5 days

> The full professional pentest report (M4) is submitted separately to the instructor. This README summarises the technical findings and evidence for portfolio purposes. Sensitive data (national ID numbers, full salary figures) has been redacted from public screenshots in this repo.

## Scope and Authorization
This engagement was conducted with written authorisation from Networkwalks on behalf of Mediroza General Hospital, strictly for educational purposes as part of the Cybersecurity Internship program. Testing was limited to the target domain only. No social engineering, denial-of-service, or testing outside the agreed scope was performed.

## Overview
This project simulates a real-world black-box penetration test against a hospital's web infrastructure, chaining multiple vulnerabilities together: authentication bypass, weak password cracking, and sensitive metadata exposure, to demonstrate the real-world impact an attacker could achieve.

## Milestone 1: Initial Access & Authentication Bypass
**Objective:** Attack the website and retrieve 3 confidential patient PDF lab reports.

### Steps
1. Conducted reconnaissance on the target website to identify exposed entry points.
2. Located the patient portal login form.
3. Tested the login's authentication mechanism for input validation weaknesses.
4. Submitted `admin' --` as the username with any value as the password.
5. The SQL query behind the login was broken by the injected `'`, and the trailing `--` commented out the rest of the query (including the password check), bypassing authentication entirely.
6. Gained access to the restricted patient portal, which listed 3 encrypted PDF pathology reports available for download.
7. Downloaded all 3 files and verified their integrity using SHA-256 checksums.

### Technical Detail
The login likely runs a query similar to:
```sql
SELECT * FROM users WHERE username='$username' AND password='$password'
```
Submitting `admin' --` turns this into:
```sql
SELECT * FROM users WHERE username='admin' --' AND password='...'
```
Everything after `--` is treated as a comment, so the password is never actually checked. The database returns a match on `username='admin'` alone.

### Result
- **Vulnerability:** SQL Injection in patient portal login form
- **Access gained:** Full login bypass, admin-level patient portal access
- **Files retrieved:** 3 encrypted PDF pathology reports (patient_report_1.pdf, patient_report_2.pdf, patient_report_3.pdf)

Screenshots: `M1-sqli-login-bypass-patient-portal.png`, `M1-patient-portal-3-pdf-files.png`, `M1-sha256-hashes-3-files.png`

## Milestone 2: Password Cracking & Data Extraction
**Objective:** Crack the encryption on all 3 retrieved files.

### Steps
1. Extracted each PDF's crackable hash using the Networkwalks Hash Calculator.
2. Ran each hash through a dictionary attack using the Networkwalks Password Cracker.
3. Files 1 and 2 cracked instantly using the built-in 100-word list. File 3 did not crack with the small wordlist, confirming the task's hint that a single approach would not work for all 3 files.
4. For file 3, downloaded the rockyou.txt wordlist (14 million real leaked passwords) and re-ran the attack with a larger dictionary, successfully cracking it.
5. Opened each PDF using its recovered password to confirm access.

### Result
| File | Password | Strength |
|---|---|---|
| patient_report_1.pdf | `123456` | Very weak, cracked instantly |
| patient_report_2.pdf | `password` | Very weak, cracked instantly |
| patient_report_3.pdf | `!@#$%^&` | Stronger, required a larger wordlist (rockyou.txt) |

Screenshots: `M2-file1-cracked-123456.png`, `M2-file2-cracked-password.png`, `M2-file3-cracked-symbols.png`

## Milestone 3: Metadata Analysis & Critical Data Exposure
**Objective:** Find the critical data exposure on the client server, and locate staff salary and shareholder data.

### Steps
1. Examined each unlocked PDF's document metadata (Title, Author, Subject, Keywords, Comments).
2. Files 1 and 2 showed standard, unremarkable metadata.
3. File 3's **Comments** field contained an internal note: *"DB backup moved to /old before site migration, do not delete"*, an unintentional information leak left in the document by staff.
4. File 3's **Author** field also showed a real staff username (`j.malik`), differing from the generic author on the other two files.
5. Navigated to `https://medirozahospital.com/old/` and found an unauthenticated, publicly accessible directory listing.
6. Downloaded the exposed file `mediroza_db_backup_2019.sql`, an unprotected internal HR/finance database backup.
7. The backup contained two tables: `staff` (30 records) and `shareholders` (10 records).

### Result
- **Vulnerability:** Sensitive metadata leakage + unauthenticated exposed directory + unprotected database backup (chained finding)
- **Staff salary data exposed:** 30 employee records including full names, job titles, departments, emails, phone numbers, **national ID numbers**, and monthly salaries (ranging from R19,000 to R160,000)
- **Shareholder data exposed:** 10 shareholder records including names, share percentages, shares held, and share class
- **Severity note:** The exposure of national ID numbers alongside salary data represents a significant privacy risk beyond the task's stated scope of "salary and shareholder details."

Screenshots (redacted): `M3-file3-metadata-clue-found.png`, `M3-exposed-directory-listing.png`, `M3-database-backup-content-REDACTED.png`

## Problems Encountered & Solutions
- **Problem:** Downloaded `rockyou.txt` from a browser source kept being silently removed after download.
  **Cause:** Likely flagged by the OS/browser as a suspicious file due to its association with password cracking.
  **Fix:** Switched to using Kali Linux's pre-installed copy of rockyou.txt (`/usr/share/wordlists/rockyou.txt`) instead of downloading it separately.
- **Problem:** John the Ripper returned "No password hashes loaded" when attempting to crack file 3's hash.
  **Cause:** Kali's default `john` build (core version) does not include PDF format support; also, a manually pasted hash risked corruption.
  **Fix:** Used John's own `pdf2john.pl` script to extract the hash directly from the PDF file, and used the Networkwalks Password Cracker with an uploaded rockyou.txt wordlist as the working alternative.
- **Problem:** Free online metadata viewer tools capped the number of free lookups per session.
  **Cause:** Usage limits on free-tier online tools.
  **Fix:** Switched between multiple online metadata viewer tools as each one's free limit was reached.

## What I Learned
- A single vulnerability is rarely the whole story: a basic SQL injection opened the door, weak passwords defeated the file encryption, and metadata left by staff leaked the path to a much larger exposure.
- Document metadata (Author, Comments) can unintentionally leak sensitive internal information, even after a file's visible content looks clean.
- Different files can require entirely different cracking approaches. It's important not to assume one method will always work.
- An exposed directory listing with no authentication is a serious finding on its own, even before considering what it contains.
- Thorough, methodical documentation at every step (screenshots, hashes, passwords found) makes writing the final report far easier than trying to reconstruct the process afterward.

## Security & Ethical Use
This assessment was performed strictly within the authorised scope of the Networkwalks Cybersecurity Internship program. These techniques must never be applied to any system without explicit written permission from the owner.

## Disclaimer
This work was done for educational purposes as part of the Networkwalks Cybersecurity Internship (Batch B083). A full professional penetration testing report, including risk ratings and remediation recommendations, was submitted separately to the instructor.
