<h1>Honeypot</h1>

<h2>Description</h2>
This project consists of 
<br />

<h2>Tools Used</h2>
- <b>Microsoft Azure</b>
- <b>Azure Sentinel</b>
- <b><a href="https://github.com/BishopFox/sliver/wiki">Sliver</a></b>

<h2>Environment</h2>

- <b>Ubuntu Server 22.04.1</b>

<h2>Project setup:</h2>
There are two machines involved in this project: a Windows 11 VM and an Ubuntu Server 22.04.1. The Windows 11 VM will be my "victim" system that has Microsoft Defender disabled, and Sysmon and LimaCharlie installed. LimaCharlie has its own Endpoint Detection & Response telemetry, as well as the capability to ship Sysmon event logs. The Ubuntu machine is going to launch the adversarial attacks. It will do this using <a href="https://github.com/BishopFox/sliver/wiki">Sliver</a>, a command-and-control (C2) framework by BishopFox.
<h2>Adversary (part 1)</h2>
I created the C2 payload (named "SPONTANEOUS_LATEX.exe") on the Ubuntu VM and downloaded it directly onto the Windows VM. After executing the payload, I could now interact directly with the C2 session on the Windows VM. Executing the commands <b>info</b> and <b>whoami</b> gives me basic information about the session and the user.
<p align="center"><img width="1278" alt="Untitled" src="https://github.com/chau-eric/LimaCharlie-Lab/assets/76719902/0aae3673-1070-4893-9086-552ab90d2c00"><br/>
</p>
More importantly, I can also see the victim's running processes using <b>ps -T</b>. Sliver automatically marks its own process in green (SPONTANEOUS_LATEX.exe) and any defensive tools in red (Sysmon).
<p align="center"><img width="259" alt="Untitled" src="https://github.com/chau-eric/LimaCharlie-Lab/assets/76719902/2463da5a-18b4-44fa-a983-2befe9f69f72"><br/>
</p>
From the Ubuntu VM, I can also execute the command <b>procdump -n lsass.exe -s lsass.dmp</b>. This is a common method to obtain a victim's OS credentials in the Windows Local Security Authority Subsystem Service (LSASS) and save it locally to your Sliver C2 server.

<h2>Detection on LimaCharlie</h2>
Using LimaCharlie to view all processes on the Windows VM, I can see the C2 payload is one of the only unsigned processes running. I can also see that it's active on the network and its source and destination IP.
<p align="center"><img width="1039" alt="Untitled" src="https://github.com/chau-eric/LimaCharlie-Lab/assets/76719902/b4baf96b-a202-4215-98a1-8d6ad61f6bc4"><br/>
</p>
The procdump command I ran earlier deals with lsass.exe, a known sensitive process. Using the <b>SENSITIVE_PROCESS_ACCESS</b> event filter, I can easily find the logs generated from the adversary.
<p align="center"><img width="1081" alt="Untitled" src="https://github.com/chau-eric/LimaCharlie-Lab/assets/76719902/b17ed47a-87c1-4b1d-8662-a8acea3a35a6"><br/>
</p>
Now that I know what the event looks like, I can create a detection & response rule to alert me anytime someone tries to access the victim's credentials again.
<p align="center"><img width="657" alt="Untitled" src="https://github.com/chau-eric/LimaCharlie-Lab/assets/76719902/100efa99-8130-409a-b835-ef36fa4fdfb6"><br/>
</p>
This rule will detect SENSITIVE_PROCESS_ACCESS events where the process ends with "lsass.exe". When detected, it will respond by generating a report called "LSASS Access."
<p align="center"><img width="1072" alt="Untitled" src="https://github.com/chau-eric/LimaCharlie-Lab/assets/76719902/ac196e3e-1454-40b5-8a53-215581b75be6"><br/>
</p>
After running the <b>procdump -n lsass.exe -s lsass.dmp</b> command again from the attacker, I can see the threat detected using the signature I just created.

<h2>Adversary (part 2)</h2>
The command I will be using this time is <b>vssadmin delete shadows /all.</b> This is a common command for attackers to use to delete Volume Shadow Copies, which would help a victim recover files in the case of a ransomware attack. It is very rare for healthy systems to need this command.
<p align="center"><img width="430" alt="Untitled" src="https://github.com/chau-eric/LimaCharlie-Lab/assets/76719902/a93c58d2-8729-4f28-87be-f75d93183d24">
<br/></p>
From my C2 session on the attacker machine, I can shell into the victim and run the command to delete all Volume Shadow Copies. I also follow it with the <b>whoami</b> command to confirm that I still have an active system shell. In this example, my victim actually doesn't have any Volume Shadow Copies, but the process is still the same.

<h2>Blocking attacks with LimaCharlie</h2>
This section will demonstrate blocking attacks instead of generating an alert. LimaCharlie's default rules can already detect <b>vssadmin delete shadows /all.</b> This lets me easily create a D&R rule off of the signature.
<p align="center"><img width="1083" alt="Untitled" src="https://github.com/chau-eric/LimaCharlie-Lab/assets/76719902/b00af5e6-af99-4b32-9f5c-8797a9035156">
  <br/></p>
<p align="center"><img width="434" alt="Untitled" src="https://github.com/chau-eric/LimaCharlie-Lab/assets/76719902/1125ff39-7e2b-4f4b-b167-7faff9d60dee">
<br/></p>
The next time it detects <b>vssadmin delete shadows /all</b> being executed, LimaCharlie will create a report in the "Detections" tab and kill the parent process that executed the command. I'll test this by executing the command again from the attacker machine.
<p align="center"><img width="413" alt="Untitled" src="https://github.com/chau-eric/LimaCharlie-Lab/assets/76719902/8a2100b7-5277-4a40-b807-955f796f3174">
<br/></p>
This time after executing the command <b>vssadmin delete shadows /all,</b> any further commands will fail to return anything. This is because the parent process has been terminated by LimaCharlie, and the system shell is no longer active. The attacker is now unable to complete their attack.

<h2>Ransomware Simulation</h2>
I will further test and refine my D&R rule using a ransomware simulator, <a href="https://github.com/NextronSystems/ransomware-simulator">Quickbuck</a>, by Nextron Systems. A successful execution of the ransomware simulation will delete all Volume Shadow Copies, create and encrypt 10,000 files, and leave a ransomware note (.txt file). Let's run it on the victim machine with my D&R rule active.
<p align="center"><img width="466" alt="Untitled" src="https://github.com/chau-eric/LimaCharlie-Lab/assets/76719902/12bd96d3-8aa4-47ed-a491-8586341c2e22">
<br/></p>
As you can see, the ransomware simulator was able to successfully complete all steps, which means my D&R rule failed to detect and kill the process. This is because the detection rule created by LimaCharlie is looking for the exact string "<b>delete shadows /all</b>" when there are other ways to execute the command. I'm going to instead refine the detection rule to look for vssadmin commands that contain "delete," "shadows," and "/all."
<p align="center"><img width="290" alt="Untitled" src="https://github.com/chau-eric/LimaCharlie-Lab/assets/76719902/027584c6-25fc-41f2-bf72-9cc248c78c7b">
<br/></p>
Now let's run the simulator again.
<p align="center"><img width="451" alt="Untitled" src="https://github.com/chau-eric/LimaCharlie-Lab/assets/76719902/3cc651e3-f474-4d3f-9299-b52aef10cd59">
<br/></p>
Now the ransomware simulator stops after deleting the Volume Shadow Copies, and I have successfully prevented a ransomware attack.
