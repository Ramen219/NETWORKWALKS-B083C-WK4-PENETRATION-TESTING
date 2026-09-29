# Penetration Testing Report: Mediroza General Hospital


|**Pentester Name**<br>**(Cybersecurity Intern)**|**Ramen Debbarma**|
|---|---|
|**Program/Batch**|B083-Networkwalks|
|**Date**|27 September 2026|
|**Modules completed**|W4 (Penetration Testing)|
|**Client/Target**|Mediroza General Hospital / `https://medirozahospital.com`|
|**Permission secured from**<br>**client?**|Yes|
|**Engagement Type:**| Full Black-Box Penetration Test|
|**Duration:**| 5 Days|

---

## 01. Executive Summary

During a 5-day black-box penetration testing engagement against Mediroza General Hospital (`https://medirozahospital.com`), Networkwalks evaluated the security posture of the target web application and supporting infrastructure. The assessment uncovered critical security vulnerabilities, including SQL Injection mechanisms in the authentication portal, weak PDF document encryption protecting sensitive medical data, and unencrypted database backup exposure.

An unauthenticated external attacker could exploit these vulnerabilities to gain unauthorized administrative access, retrieve confidential patient medical reports, and extract internal corporate records including employee national IDs, salaries, and shareholder details. Immediate remediation steps are outlined in this report to secure the target infrastructure.

---

## 02. Scope and Methodology

### Scope
* **Target Domain:** `https://medirozahospital.com`.
* **Testing Type:** Full Black-Box Penetration Test.
* **Rules of Engagement:** Testing limited strictly to the target domain; no social engineering; no denial of service (DoS); no testing outside agreed parameters.

### Tools and Technologies Used
* **Burp Suite Professional:** Request interception, parameter manipulation, and SQL injection testing on `/staff/login.php` and `/patient/login.php` endpoints.
* **Gobuster / Cyscan.io:** Automated web directory fuzzing, endpoint discovery, and hidden directory enumeration.
* **theHarvester:** Open Source Intelligence (OSINT) gathering for domain target profiling.
* **pdf2john & Hashcat:** Extracting hashes from encrypted PDF files and performing dictionary-based password recovery against `rockyou.txt`.
* **MySQL Dump Analysis:** Manual analysis and parsing of unencrypted SQL database backup files.

---

## 03. Findings and Proof of Exploitation

### Milestone 1: Initial Access & SQL Injection
* **Vulnerability:** SQL Injection (SQLi) / Authentication Bypass
* **Description:** Initial endpoint discovery via `theHarvester` and `Cyscan.io` mapped target login interfaces at `/staff/login.php` and `/patient/login.php`. Submitting a classic SQL injection vector (`' OR 1=1 --`) in Burp Suite Repeater triggered verbosely unhandled MySQL syntax errors (`mysql_query()`).
* **Exploitation & Evidence:** Bypassing authentication yielded direct access to the restricted patient portal (`/patient/portal.php`), exposing three password-protected patient pathology reports (`patient_report_1.pdf`, `patient_report_2.pdf`, `patient_report_3.pdf`).

#### Images & Descriptions (Milestone 1)

![Reconnaissance via theHarvester and Cyscan.io](/Screenshots/image3.jpg)
* **Image Description:** Displays the OSINT reconnaissance phase using `theHarvester` in terminal alongside `Cyscan.io` endpoint analysis showing discovered paths `/staff/login.php` and `/patient/login.php` marked as high priority.

![SQL Injection Testing on Patient Portal](/Screenshots/image1.jpg)
* **Image Description:** Burp Suite Repeater manipulating the HTTP POST request to `/staff/login.php` with `' OR 1=1 --`, triggering a raw MySQL database syntax error box in the Patient Portal web page UI.

![Successful Patient Portal Authentication Bypass](/Screenshots/image2.jpg)
* **Image Description:** Captures successful authentication bypass in Burp Suite Repeater, returning an HTTP `302 Found` response and displaying the "My lab reports" dashboard containing download links for three encrypted pathology reports.

---

### Milestone 2: Data Extraction & PDF Encryption Cracking
* **Vulnerability:** Weak Password Protection on Patient Documents
* **Description:** The downloaded patient lab reports were encrypted with PDF passwords. Hashes were extracted using `pdf2john` (`hash1.txt`, `hash2.txt`) and subjected to dictionary attacks using `Hashcat` with `rockyou.txt`.
* **Exploitation & Evidence:** The encryption on all three files was broken within seconds, revealing weak passwords (such as `123456`) and granting full access to confidential medical lab reports for patients S. Dlamini, P. Reddy, and E. Thompson.

#### Images & Descriptions (Milestone 2)

![Extracting Hashes with pdf2john](/Screenshots/image8.jpg)
* **Image Description:** Terminal output executing `pdf2john patient_report_2.pdf > hash2.txt`, followed by inspecting `cat hash1.txt` and `cat hash2.txt` to isolate the PDF $pdf$2*3*128 encryption strings.

![Hashcat Attack Execution](/Screenshots/image4.jpg)
* **Image Description:** Demonstrates `Hashcat` executing Mode `10500` (PDF 1.4 - 1.6) against `hash1.txt` using `/usr/share/wordlists/rockyou.txt`, recovering the plaintext password `123456` in 1 second.

![Cracked Patient Report 1 (S. Dlamini)](/Screenshots/image5.jpg)
* **Image Description:** Displays Hashcat cracking `patient_report_1.pdf` alongside the decrypted Pathology Laboratory Report document for patient **S. Dlamini**.

![Cracked Patient Report 2 (P. Reddy)](/Screenshots/image7.jpg)
* **Image Description:** Depicts Hashcat output cracking `patient_report_2.pdf` at 77,376 H/s and rendering the decrypted lab report for patient **P. Reddy**.

![Cracked Patient Report 3 (E. Thompson)](/Screenshots/image6.jpg)
* **Image Description:** Displays terminal processes running alongside the opened decrypted Pathology Laboratory Report for patient **E. Thompson**.

---

### Milestone 3: Critical Data Exposure (Database Backup Disclosure)
* **Vulnerability:** Exposed Backup File / Insecure Direct Object Reference
* **Description:** Directory brute-forcing using `Gobuster` identified an unindexed `/old/` web path containing a legacy database file: `/old/mediroza_db_backup_2019.sql`.
* **Exploitation & Evidence:** Fetching the `.sql` dump file exposed the internal database schema (`mediroza_hr`) containing full names, designations, department details, email addresses, phone numbers, national IDs, monthly salaries (ZAR), and shareholder records.

#### Image & Description (Milestone 3)

![Gobuster Discovery and SQL Backup Dump](/Screenshots/image9.jpg)
* **Image Description:** Shows `Gobuster v3.8.2` directory brute-forcing revealing status code `301` for `/old/`, alongside the opened raw SQL dump `mediroza_db_backup_2019.sql` containing internal staff table structures, salaries, and personal national IDs.

---

## 04. Risk Rating

| Finding / Vulnerability | Severity | Impact Summary |
| :--- | :--- | :--- |
| **SQL Injection (Authentication Bypass)** | **Critical** | Grants unauthorized administrative access to patient portal dashboards and records. |
| **Sensitive File Disclosure (`.sql` Backup)** | **Critical** | Exposes private employee PII, national ID numbers, monthly salary structures, and corporate shareholder details. |
| **Weak PDF Password Encryption** | **High** | Allows rapid offline dictionary cracking of confidential patient pathology files. |
| **Information Disclosure via Legacy Endpoints** | **Medium** | Exposed `/old/` and `/staff/` directories increase attack surface visibility. |

---

## 05. Recommendations and Remediation

1. **Implement Parameterized Queries:** Update all login scripts (`/staff/login.php`, `/patient/login.php`) to use prepared statements and parameterized inputs to completely mitigate SQL injection risks.
2. **Remove Publicly Accessible Backup Files:** Immediately delete or relocate all `.sql` database dumps, legacy assets, and old web directories (`/old/`) out of the web server document root.
3. **Upgrade Document Security Mechanisms:** Replace weak static PDF passwords with secure, authenticated session-based document viewing portals or dynamic multi-factor access tokens.
4. **Harden Directory Access Control:** Disable web server directory listing and implement explicit Access Control Lists (ACLs) to restrict access to internal management routes.


-End-

***Author***  
**Ramen Debbarma**  
**Cybersecurity Intern**  
LinkedIn: [Ramen Debbarma](https://www.linkedin.com/in/ramen-debbarma-a71632286/)

**Project Information**  
**Program Name:** Cybersecurity program at Networkwalks | **Week:** 04 | **Repository:** GitHub
