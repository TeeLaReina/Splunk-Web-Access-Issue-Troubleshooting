# Splunk Web Access Issue – Troubleshooting & Resolution (pfSense + VirtualBox SOC Lab)
Decided to give you a peak into my mind. I encountered an issue accessing Splunk whilst configuring my home SOC lab and decided to share how I was able to fix it. Let's dive in!


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

### The Problem

Splunk was installed and running on the Ubuntu Server, but:
- I could not access Splunk Web from my browser
- SSH access was inconsistent
- I could ping some systems but still couldn’t reach services
- Sometimes access only worked when I changed network interfaces manually


At first it looked like a Splunk problem nuy it turned out to be a networking and firewall problem.

Actually… several of them.😅

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


### What I Checked First

Before assuming Splunk was broken, I checked:
- Is Splunk actually running?

Command: sudo /opt/splunk/bin/splunk status

Result:
- splunkd is running
- splunk helpers are running

Good. So Splunk itself was not the issue.

### Troubleshooting Process

### Issue 1 – Port 8000 Was Blocked by Ubuntu Firewall

Even though Splunk was running, Ubuntu’s firewall (UFW) was blocking external access to the web interface. When I ran http://192.168.42.103:8000 in my browser, the result was nothing useful as I still had no access. 

#### What Fixed It

Command: sudo ufw allow 8000/tcp

Why, you might ask:
Well, Splunk Web runs on TCP port 8000 by default so if UFW blocks that port, the browser cannot reach the service even though it was runnig on port 8000. So after allowing the port, Splunk Web became reachable.

### Issue 2 – IP Address Conflict on OPT1

This one was sneaky and took about 3 days to figure out. (Don't judge, I'm not a CompSci student and I only started learning about a year ago)

Anyways, the pfSense OPT1 interface and the VirtualBox host-only adapter were both using the same IP address:

192.168.56.1

That created a routing conflict because the system didn’t know which one should respond. Which is why ping sometimes worked but browser access failed.

#### What Fixed It

I changed the pfSense OPT1 interface IP to:

192.168.56.2 (so only the VirtualBox host-only adapter was using 192.168.56.1)

Why did I do that:

Well, two devices on the same network should not share the same IP. That conflict is exactly what was breaking routing.

After changing it:
- Browser access improved, and
- Routing became stable

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

Why it failed:
This only allowed traffic to the LAN interface itself, not beyond it. So devices couldn’t properly route traffic through pfSense to the internet.

#### What Fixed It?

I updated the rule to:

Source: LAN subnet
Destination: any

Why:
- pfSense handles the routing.
- The LAN rule should allow traffic out, while WAN handles internet access.

So after fixing this:
- Internet access returned, and
- VM connectivity improved

### Commands That Helped During Troubleshooting

##### Check IP Address

Command: ip a

I used this to confirm interface IPs and detect conflicts

##### Check Routing Table

Command: ip route show

Used this to to verify default gateway settings

##### Test Connectivity

I did an unhealthy amount of pinging and this helped A LOT.
ping 192.168.42.1
ping 8.8.8.8
ping google.com

Used ping to test:
- LAN access
- Internet access, and
- DNS resolution
  
##### Check UFW Rules

Command: sudo ufw status

I used this on the Ubuntu server to confirm blocked/open ports

##### Splunk Service Check

Command: sudo /opt/splunk/bin/splunk status

This particular command hepled verify if Splunk was working because you definitely do not want to be trubleshooting when the  server you are troubleshooting is offf. I guess that might be very bad😅.

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

### But Why Exactly did the Final Fix Work?

I guess the problem was never Splunk itself becasue Splunk was installed correctly. The real issue was the fact that the network path to reach it was broken. The final fix worked because it solved all three layers:
- Application Layer: Splunk was running
- Host Layer: Ubuntu firewall allowed port 8000
- Network Layer: pfSense routing and interface communication were corrected

Once all three matched, Splunk became accessible exactly as expected.

Sometimes the issue isn’t the tool. Sometimes the issue is everything around the tool, like the firewall rules or even misconfiguratoin.

### What I Learned

This issue taught me way more than just "how to access Splunk".

I learned:
- The difference between routing and firewall filtering.
- How pfSense actually handles traffic between interfaces, not just theory.
- Why host-only and internal networks behave differently.
- How Linux firewalls can silently block services.
- Why troubleshooting is one of the most important security skills.

Honestly, setting up the tools is easy but figuring out why they refuse to work is where the real learning happens.

Hi, I'm Duze Yetunde (TeeLaReina), an aspiring SOC Analyst who's curious about systems and securing them. 
