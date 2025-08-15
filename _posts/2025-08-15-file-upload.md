---
title: "Unrestricted File Upload"
date: 2025-08-15 12:00:00 +0530
categories: [Web Security, Write-ups]
tags: [file-upload]
toc: true
---

## File upload vulnerability

Given how common file upload functions are — from updating profile pictures to sharing videos — almost every modern application uses them. But it’s not as simple as just accepting any file a user uploads.

**File upload vulnerabilities** occur when a web application fails to properly validate and handle uploads, allowing attackers to make malicious file uploads.

In short, if a web server allows users to upload files without sufficiently validating things like their **name**, **type**, **contents**, or **size**, it can lead to **remote code execution**, **file overwriting**, or even **denial-of-service attacks**. This might even include server-side scripts capable of executing code on the server.

From a security perspective, the worst possible scenario is when a website allows you to upload server-side scripts — such as PHP, Java, or Python files — and is also configured to execute them as code. This makes it trivial to create your own web shell on the server.

---

In the application I was testing, the upload function was intended to accept only PDF files — for example, for users to upload reports and invoices in `.pdf` format. However, the application did not properly validate the file type on the server side. It likely relied only on the filename extension, which can be easily spoofed.

I first tested by uploading a legitimate PDF, then intercepted the request using Burp Suite Proxy then replaced the PDF content with a simple PHP payload:
```php
<?php echo system($_GET['cmd']); ?>
```
This script takes the content of the parameter cmd and executes it. The server accepted the file without any validation errors.

## Secure File Upload Best Practices

### Extension Validation
- Check file extensions after decoding filenames.
- Watch for bypass attempts like double extensions (`.jpg.php`) or null byte tricks (`.php%00.jpg`).
- Only permit business-critical file types. For images stick to one agreed-upon format and for documentsvlimit to essential types like PDF, DOCX, JPG.


### Content Validation
- Never trust user-provided `Content-Type` headers.
- Dont just trust just the file extension (.pdf, .jpg, etc.) or the Content-Type header sent by the browser. Instead check the actual binary signature inside the file,magic bytes to see what the file really is.
- Use whitelisting over blacklisting.
- Avoid ZIP uploads if possible due to inherent security risks.


### Storage Security
- Store files on a separate host if possible.
- Keep files outside the web root.
- If in web root, strictly limit write permissions.

### Access Controls
- Require authentication for uploads.
- Implement proper authorization checks to prevent bypass.
- Set restrictive filesystem permissions.
- Enforce upload/download size limits.

### Filename Security
- Generate random names (UUID/GUID) when possible.
- If keeping user filenames:
  - Limit length.
  - Allow only alphanumeric characters, hyphens, spaces, and periods.
  - Block hidden files and directory traversal attempts.
---

Even simple file upload functions, like allowing PDFs, can become a major attack vector if server-side validation is not enforced. By following these best practices, you drastically reduce the risk of file uploads becoming an entry point for attackers.

