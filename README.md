# Splunk Web Access Issue – Troubleshooting & Resolution (pfSense + VirtualBox SOC Lab)
I encountered an issue accessing Splunk whilst configuring my home SOC lab and decided to share how I was able to fix it. This should help you get a good grasp of how I attack issues and troubleshoot. Let's dive in!


## Project Context

This issue happened while building my main project:

<a href="https://github.com/TeeLaReina/End-to-End-SOC-Lab-Simulation-using-Splunk-pfSense-and-Virtual-Machines">End-to-End SOC Operations Simulation: Detection, Triage & Reporting</a>

The goal of the main project is to build a functional home SOC lab that simulates real-world security operations using:
- pfSense as the firewall/router
- Splunk as the SIEM
- Ubuntu Server as the Splunk server
- VirtualBox for virtualization
- Segmented networks (LAN + OPT1) for better security architecture

But before getting to alert monitoring and incident triage, I needed the infrastructure to actually work first.

That was where the problem started.🤦‍♀️

Splunk was installed successfully on my Ubuntu Server VM, but I could not access the Splunk Web Interface from my SOC analyst machine.

Instead of:

http://192.168.42.103:8000

…I got nothing.

No dashboard. No login page. Just stress.

This project documents how I investigated and resolved that issue so let's get into it.

<br><br>

### The Problem

Splunk was installed and running on the Ubuntu Server, but:
- I could not access Splunk Web from my browser
- SSH access was inconsistent
- I could ping some systems, but still couldn’t reach services
- Sometimes, access only worked when I changed network interfaces manually


At first, it looked like a Splunk problem, but it turned out to be a networking and firewall problem.

Actually… several of them.😅

<br><br>

### Lab Setup

First, here's an overview of the core components so you can get a good idea of the issues.
#### pfSense Firewall, used as the main router and traffic controller
- WAN for Internet access
- LAN for the Internal servers + vulnerable machines
- OPT1 for the SOC analyst monitoring network

#### Ubuntu Server VM, used as the Splunk Server
- Connected to LAN
- Running Splunk Enterprise
  
#### SOC Analyst Host, my main monitoring system
- Connected to OPT1
- Used to access Splunk Web and monitor logs

#### VirtualBox, used to create and manage the segmented virtual network environment

<br><br>

### What I Checked First

Before assuming Splunk was broken, I checked:
- Is Splunk actually running?

Command: <i>sudo /opt/splunk/bin/splunk status</i>

Result:
- splunkd is running
- splunk helpers are running

Good. So Splunk itself was not the issue.

<br><br>

### Troubleshooting Process

### Issue 1 – Port 8000 Was Blocked by the Ubuntu Firewall

Even though Splunk was running, Ubuntu’s firewall (UFW) was blocking external access to the web interface. When I ran http://192.168.42.103:8000 in my browser, the result was nothing useful, as I still had no access. 

#### What Fixed It

Command: <i>sudo ufw allow 8000/tcp</i>

Why, you might ask:
Well, Splunk Web runs on TCP port 8000 by default, so if UFW blocks that port, the browser cannot reach the service even though it was running on port 8000. So after allowing the port, Splunk Web became reachable.

<br><br>

### Issue 2 – IP Address Conflict on OPT1

This one was sneaky and took about 3 days to figure out. (Don't judge, I'm not a CompSci student and I only started learning about a year ago)

Anyways, the pfSense OPT1 interface and the VirtualBox host-only adapter were both using the same IP address:

192.168.56.1

That created a routing conflict because the system didn’t know which one to respond to. Which is why ping sometimes worked, but browser access failed.

<figure>
  <img src="https://github.com/TeeLaReina/Splunk-Web-Access-Issue-Troubleshooting/blob/main/Images/the-virtualbox-host-only-network-adadpter-ap-addess-192.168.56.1-that-conflicted-eith-the-pfsense-opt1-interface.png" alt="https://github.com/TeeLaReina/Splunk-Web-Access-Issue-Troubleshooting/blob/main/Images/the-virtualbox-host-only-network-adadpter-ap-addess-192.168.56.1-that-conflicted-eith-the-pfsense-opt1-interface" style="width: 50%; height: auto;">
  </img>
  <figcaption>
    <sub>The host-only adapter network (vboxnet0) showing the conflicting IP address - 192.168.56.1</sub>
  </figcaption>
</figure>

<br><br>

<figure>
  <img src="https://github.com/TeeLaReina/Splunk-Web-Access-Issue-Troubleshooting/blob/main/Images/the-opt1-address-for-my-pfsense-router-that-conflicted-with-the-soc-analyst-system-192-168-56-1.png.png" alt="https://github.com/TeeLaReina/Splunk-Web-Access-Issue-Troubleshooting/blob/main/Images/the-opt1-address-for-my-pfsense-router-that-conflicted-with-the-soc-analyst-system-192-168-56-1.png.png" style="width: 50%; height: auto;">
  </img>
  <figcaption>
    <sub>The opt1 interface on my pfSense router showing the conflicting IP address - 192.168.56.1 as well</sub>
  </figcaption>
</figure>

#### What Fixed It

I changed the pfSense OPT1 interface IP from 192.168.6.1 to:

192.168.56.2 (so only the VirtualBox host-only adapter was using 192.168.56.1)

<figure>
  <img src="https://github.com/TeeLaReina/Splunk-Web-Access-Issue-Troubleshooting/blob/main/Images/changed-the-ip-address-that-conflicted-with-the-host-only-adapter-network-to-192-168-56-2-pfsense.png" alt="https://github.com/TeeLaReina/Splunk-Web-Access-Issue-Troubleshooting/blob/main/Images/changed-the-ip-address-that-conflicted-with-the-host-only-adapter-network-to-192-168-56-2-pfsense.png" style="width: 50%; height: auto;">
  </img>
  <figcaption>
    <sub>Changed the IP address on my pfSense opt1 router interface from 192.168.56.1 to 192.168.56.2</sub>
  </figcaption>
</figure>

#### Why did I do that:

Well, two devices on the same network should not share the same IP. That conflict is exactly what was breaking routing.

After changing it:
- Browser access improved, and
- Routing became stable

<br><br>

### Issue 3 – Wrong LAN Firewall Rule

At one point, I restricted LAN traffic too much and I didn't even know this was a bad thing, at least not until my project. I set the rule to only allow traffic to:

192.168.42.1/24

which was too limited.

That broke internet access and general communication. My LAN devices basically started acting like they were on punishment.

#### So What Didn’t Work

Here's the firewall rule:

Allow
Source: LAN subnet

Destination: 192.168.42.1/24

#### Why it failed:
This only allowed traffic to the LAN interface itself, not beyond it. So devices couldn’t properly route traffic through pfSense to the internet.

#### What Fixed It?

I updated the rule to:

Source: LAN subnet

Destination: any

Why:
- pfSense handles the routing.
- The LAN rule should allow traffic out, while the WAN handles internet access.

So after fixing this:
- Internet access returned, and
- VM connectivity improved

<br><br>

### Issue 4 – Inability to SSH into the Splunk Ubuntu Server

This wasn't the last issue I encountered, mind you, but at this point, after resolving the IP address conflicts and many firewall issues, I was unable to SSH into the Splunk interface (192.168.42.103)

This meant that if I needed to update anything on the Splunk server itself, I couldn't do it. I know you might be thinking, 'Well, you had access to the server, so why not just tweak and resolve from there?'. 

My response to you: In a standard SOC environment, direct access to the server, any server, is very limited for security reasons. So NO! I can't just tweak from the server directly; I have to SSH into it.

#### So What Didn’t Work

Clearly, SSH.

#### What Fixed It?

You might wanna sit tight for this one!

1. The first thing I did was to make sure that SSH was active on the Splunk Ubuntu Server by running <i>systemctl status ssh</i>.
2. Since SSH was running, I proceeded to make sure it was actually listening on port 22 using <i>ss -tulnp | grep 22</i>, and we can see that it has a process ID of 4096, listening across all interfaces (0:0:0:0)
3. Finally, I pinged <i>192.168.42.1</i> to ascertain that the server could actually access the LAN interface, which it could.
All this led me to believe that the issue wasn't coming from the server, but rather, from my SOC Host.

<figure>
  <img src="https://github.com/TeeLaReina/Splunk-Web-Access-Issue-Troubleshooting/blob/main/Images/ssh-was-active-on-the-splunk-ubuntu-server.png" alt="https://github.com/TeeLaReina/Splunk-Web-Access-Issue-Troubleshooting/blob/main/Images/ssh-was-active-on-the-splunk-ubuntu-server.png" style="width: 50%; height: auto;">
  </img>
  <figcaption>
    <sub>Troubleshooting on the Splunk Server VM first</sub>
  </figcaption>
</figure>

<br><br>

On getting to my SOC host System, here's how troubleshooting went:
1. I ran the <i>arp -a</i> command to really see why gateways my SOC host system could reach
2. I ran <i>ip route 192.168.42.103</i> to see which interface on my SOC host system was trying to access the Ubuntu Splunk Server
3. Once I figured out that my SOC host system was trying to connect with 192.168.42.103 via the wrong interface, I updated it using <i>sudo ip route add</i>
4. I then tried to ping 192.168.42.103 again, and finally, it worked!
  
<figure>
  <img src="https://github.com/TeeLaReina/Splunk-Web-Access-Issue-Troubleshooting/blob/main/Images/use-a-breakdown-of-the-commandds-i-used-to-resolve-the-ssh-into-splunk-ubuntu-server-issue-1.png" alt="https://github.com/TeeLaReina/Splunk-Web-Access-Issue-Troubleshooting/blob/main/Images/use-a-breakdown-of-the-commandds-i-used-to-resolve-the-ssh-into-splunk-ubuntu-server-issue-1.png" style="width: 50%; height: auto;">
  </img>
  <figcaption>
    <sub>The commands I used to resolve the SSH issue</sub>
  </figcaption>
</figure>

<br><br>

Why:
- Our systems have different interfaces. If the wrong one tries to communicate to a certain IP address, it won't work. Liken the interfaces to keys and the IP addresses to doors. If you use key-A for door-B (IP address B), it can never work.

So after fixing this:
- I was able to SSH into my Splunk Ubuntu Server, and
- I can easily access the Splunk server.

  <br><br>

### Commands That Helped During Troubleshooting

##### Check IP Address

Command: <i>ip -a</i>

I used this to confirm interface IPs and detect conflicts

##### Check Routing Table

Command: <i>ip route show</i>

Used this to verify default gateway settings

##### Test Connectivity

I did an unhealthy amount of pinging and this helped A LOT.
ping 192.168.42.1
ping 8.8.8.8
ping google.com

Used <i>ping</i> to test:
- LAN access
- Internet access, and
- DNS resolution
  
##### Check UFW Rules

Command: <i>sudo ufw status</i>

I used this on the Ubuntu server to confirm blocked/open ports

##### Splunk Service Check

Command: <i>sudo /opt/splunk/bin/splunk status</i>

This particular command helped verify if Splunk was working because you definitely do not want to be troubleshooting when the  server you are troubleshooting is off. I guess that might be very bad😅.

<br><br>

### Final Result

#### After fixing:
- the UFW port blocking
- the IP address conflict, and
- the incorrect pfSense firewall rules


I was able to successfully:
- Access Splunk Web 
- Keep network segmentation intact
- Maintain secure communication between networks, and
- Continue building the actual SOC lab

Peace was restored (finally).

<br><br>

### But Why Exactly did the Final Fix Work?

I guess the problem was never Splunk itself because Splunk was installed correctly. The real issue was the fact that the network path to reach it was broken. The final fix worked because it solved all three layers:
- Application Layer: Splunk was running
- Host Layer: Ubuntu firewall allowed port 8000
- Network Layer: pfSense routing and interface communication were corrected

Once all three matched, Splunk became accessible exactly as expected.

Sometimes the issue isn’t the tool. Sometimes the issue is everything around the tool, like the firewall rules or even misconfiguration.

<br><br>

### What I Learned

This issue taught me way more than just "how to access Splunk".

I learned:
- The difference between routing and firewall filtering.
- How pfSense actually handles traffic between interfaces, not just theory.
- Why host-only and internal networks behave differently.
- How Linux firewalls can silently block services.
- Why troubleshooting is one of the most important security skills.

Honestly, setting up the tools is easy, but figuring out why they refuse to work is where the real learning happens.

Hi, I'm Duze Yetunde (TeeLaReina), an aspiring SOC Analyst who's curious about systems and securing them. 
