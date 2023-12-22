<h1>Honeypot (Microsoft Sentinel Lab)</h1>

<h2>Description</h2>
This project consists of setting up a VM, configuring Log Analytics Workspace, and configuring a Sentinel workbook in Microsoft Azure. The end result is a map that displays the geolocation of incoming RDP brute force attacks on a world map.
<br />

<h2>Tools Used</h2>

- <b>Microsoft Azure</b>
- <b>PowerShell</b>
- <b><a href="https://ipgeolocation.io">ipgeolocation</a></b>

<h2>Project demonstration:</h2>
<p align="center"><img width="1146" alt="Untitled1" src="https://github.com/chau-eric/honeypot/assets/76719902/f232c1bf-dc42-4a0c-bd30-99b0689aec4b">
</p>
World map of incoming attacks after approximately 1 hour. Metrics along the bottom represent the number of attacks from each IP.
<br/>
<h2>VM setup</h2>
The virtual machine that will be receiving all of the attacks is a Windows 10 Pro VM setup in Microsoft Azure. It has a custom firewall to allow all traffic from the internet into the VM.

 using <a href="https://ipgeolocation.io">ipgeolocation's free API.</a>
