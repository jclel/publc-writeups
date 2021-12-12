## Day 10

> McSkidy is trying to discover how the attackers managed to penetrate the network and cause damage to the Best Festival Company’s infrastructure. She decided to start doing a security assessment of her systems to discover how Grinch Enterprises managed to cause this damage. She started by conducting a security assessment of her systems to discover how Grinch Enterprises managed to cause this damage and wonders what service they exploited.

We're getting into the core class now for any beginner's course on pentesting: nmap. You've seen it in The Matrix, now it's time to use it yourself. Nmap is probably the most important tool in the pentester's toolkit. There's a really nice intro to port scanning on today's TryHackMe writeup that's worth a look, explaining what exactly *is* a port and what nmap is doing. I won't retread here. Let's just get into it.  

>  Help McSkidy and run nmap -sT 10.10.184.117. How many ports are open between 1 and 100? 

Let's run the scan and see what's open:

    Starting Nmap 7.91 ( https://nmap.org ) at 2021-12-10 19:59 EST
    Nmap scan report for 10.10.184.117
    Host is up (0.28s latency).
    Not shown: 998 closed ports
    PORT   STATE SERVICE
    22/tcp open  ssh
    80/tcp open  http

We can see two ports: the server's running SSH and a web server. Looking at this I'd immediately try the website and try for vulnerabilities there, hoping to either exploiit it for a reverse shell or attempting to find login details I could use to get onto the machine via SSH.  

> Now run nmap -sS 10.10.184.117. Did you get the same results? (Y/N)

We have to run the SYN scan with `sudo` so it should actually be `sudo nmap -sS 10.10.184.117`. We get exactly the same results.  

> If you want Nmap to detect the version info of the services installed, you can use nmap -sV 10.10.184.117. What is the version number of the web server?

Now we get different output:

    22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
    80/tcp open  http    Apache httpd 2.4.49

If you want to be absolutely sure about the version, you can also pass the `--version-intensity` flag to `nmap` and set it between 0 and 9 (the default is 7.) You can find out more about that [here](https://nmap.org/book/man-version-detection.html) which also includes this funny tidbit:

> By default, Nmap version detection skips TCP port 9100 because some printers simply print anything sent to that port, leading to dozens of pages of HTTP GET requests, binary SSL session requests, etc.

Imagine port scanning a network and accidentally tipping them off because their printer starts going mad. Oops!  

> By checking the vulnerabilities related to the installed web server, you learn that there is a critical vulnerability that allows path traversal and remote code execution. Now you can tell McSkidy that Grinch Enterprises used this vulnerability. What is the CVE number of the vulnerability that was solved in version 2.4.51?

A CVE number is how vulnerabilities are identified. They're always in the format CVE-[Year]-[ID]. For this vulnerability, it's CVE-2021-42013.  

> You are putting the pieces together and have a good idea of how your web server was exploited. McSkidy is suspicious that the attacker might have installed a backdoor. She asks you to check if there is some service listening on an uncommon port, i.e. outside the 1000 common ports that Nmap scans by default. She explains that adding -p1-65535 or -p- will scan all 65,535 TCP ports instead of only scanning the 1000 most common ports. What is the port number that appeared in the results now?

Nmap can sometimes be pretty slow to scan all 65535 ports, so I prefer to use [rustscan](https://github.com/RustScan/RustScan) and then pipe the ports it finds to nmap:

    rustscan -a 10.10.184.117 -r 0-65535 --ulimit 5000 -b 1250 -- -Pn -sC -sV -vvvv  

It took `rustscan` about 5 seconds to find the port necessary (20212). Nmap's estimated time to scan all the ports? 20 minutes.  

> What is the name of the program listening on the newly discovered port?

Let's look at our output:

    20212/tcp open  telnet  syn-ack Linux telnetd

We can connect to that port:

    telnet 10.10.184.117 20212

And we get:

    Trying 10.10.184.117...
    Connected to 10.10.184.117.
    Escape character is '^]'.
    Ubuntu 20.04.3 LTS
    bestfestival-www login:

Wow, straight up login prompt for the server. 