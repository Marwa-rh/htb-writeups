# htb-writeups
Hack The Box: Oopsie Walkthrough

Note: This is a walkthrough documenting my exact steps and thought process while solving the Oopsie machine on Hack The Box. In accordance with HTB policy, active flags are redacted.

📌 Target Summary

Machine Name: Oopsie

OS: Linux

Difficulty: Easy (Starting Point - Tier 2)

IP Address: 10.129.X.X

Focus Areas: Web Enumeration, IDOR, Reverse Shells, Privilege Escalation (SUID & PATH Hijacking)

🎯 Summary of What I Did

In this lab, I started by scanning open ports to figure out what services were running. After finding a web server, I explored the site and intercepted HTTP traffic using Burp Suite. I noticed a parameter vulnerability (IDOR) that allowed me to view admin user data and steal access keys. I then used those admin privileges to upload a PHP reverse shell, getting my first shell access. From there, I dug around the file system, found database credentials in plain text, switched to another user, and finally hijacked a misconfigured SUID binary's execution path to get full root access.

1. Reconnaissance & Initial Enumeration

Step 1: Scanning the Target

First, I ran an nmap scan against the machine's IP address to see what ports were open and what services were listening.

nmap -sC -sV -oA nmap/oopsie 10.129.X.X



What I found:

Port 22 (SSH): Open, running OpenSSH 8.2p1.

Port 80 (HTTP): Open, running an Apache web server.

Figure 1: My initial Nmap scan results.

Step 2: Exploring the Web Application

Since Port 80 was open, I opened my browser and navigated to http://10.129.X.X. I saw a standard company website.

I wanted to find hidden directories or extra endpoints that weren't visible on the main page, so I ran gobuster to perform directory brute-forcing:

gobuster dir -u http://10.129.X.X -w /usr/share/wordlists/dirb/common.txt



While Gobuster was running, I also clicked around the web pages and noticed an admin login link and dynamic URL parameters like ?active=admin.

Figure 2: Directory enumeration with Gobuster.

2. Gaining Access (Foothold)

Step 3: Finding and Exploiting the IDOR

Next, I logged into the portal as a guest/regular user and opened Burp Suite to intercept my traffic. While looking at the account page, I saw a request containing a user ID parameter: ?id=1.

I decided to test for Insecure Direct Object Reference (IDOR) by changing the id values manually (?id=2, ?id=3, etc.) to see if the server would let me view other users' data without permission.

Sure enough, when I changed the ID to 34658, the server returned the admin user's account details, including their access key (8832)!

Figure 3: Intercepting the HTTP request in Burp Suite and modifying the user ID.

Step 4: Uploading a Web Shell

Now that I had the admin credentials and super admin key, I logged into the administrative interface. I immediately went looking for a place where I could upload files.

I found an upload page, but I wanted to make sure I could execute code on the server. So, I prepared a basic PHP reverse shell script, uploaded it, and navigated to /uploads/shell.php to execute it.

Before triggering the script in the browser, I set up a Netcat listener on my terminal:

nc -lvnp 9001



As soon as I loaded the shell URL in my browser, my Netcat listener caught the incoming connection, and I was in as www-data!

Figure 4: Catching my reverse shell in Netcat as www-data.

3. Privilege Escalation

Step 5: Escalating to User (robert)

Once I had my initial shell, I started enumerating local files to see what else was on the box. I navigated to the web server root directory (/var/www/html) and began reading configuration files.

Inside /var/www/html/cdn-cgi/login/db.php, I noticed plain-text database credentials:

cat /var/www/html/cdn-cgi/login/db.php



I checked /etc/passwd to see valid users on the system and saw a user named robert. I tried using the database password I found to log in as robert, and it worked!

su robert



🚩 User Flag: HTB{********************************}

Step 6: Escalating to root via SUID PATH Hijacking

Now that I was logged in as robert, I wanted to get full root control. I checked what groups robert belonged to and found he was part of the bugtracker group.

I looked for binaries associated with this group and found /usr/bin/bugtracker, which had the SUID bit set (meaning it runs with the privileges of its owner, root).

ls -la /usr/bin/bugtracker



I tested running the binary to see what it did. I noticed that when it executed, it called the system utility cat without specifying its full path (it called cat instead of /bin/cat).

Since it relied on the system $PATH to find cat, I realized I could perform PATH Hijacking:

I created my own malicious script named cat inside /tmp that simply launches /bin/bash.

I made it executable with chmod +x.

I added /tmp to the very beginning of my $PATH environment variable.

I ran /usr/bin/bugtracker again.

# Step-by-step commands I executed:
echo "/bin/bash" > /tmp/cat
chmod +x /tmp/cat
export PATH=/tmp:$PATH

# Running the SUID binary
/usr/bin/bugtracker



Because /tmp was now first in my $PATH, the SUID binary executed my custom /tmp/cat script as root, instantly spawning a root shell!

Figure 5: Successfully getting a root shell via SUID PATH hijacking.

🚩 Root Flag: HTB{********************************}

💡 What I Learned & Key Takeaways

Always check access controls on parameters: Seeing numbers like ?id=1 in URLs or request bodies is a huge red flag for IDOR vulnerabilities.

Never hardcode credentials in source code: Finding DB passwords inside /var/www/html made user escalation trivial.

Always use full paths in SUID binaries: Developers should always specify /bin/cat instead of cat when invoking system tools in compiled programs to prevent PATH manipulation attacks.

📂 Folder Structure

.
├── README.md
└── assets/
    ├── nmap-scan.png
    ├── gobuster-results.png
    ├── idor-exploit.png
    ├── reverse-shell.png
    └── root-escalation.png

