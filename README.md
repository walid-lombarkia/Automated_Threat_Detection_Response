<h1> Automated_Threat_Detection_Response </h1>

 <h2>Description</h2>
This project integrates Wazuh, Shuffle, and TheHive to automate threat detection and response. Wazuh monitors security events and detects anomalies, triggering alerts. Shuffle enriches and processes incidents, forwarding them to TheHive for investigation. Automated response actions mitigate threats, enhancing SOC efficiency.<br/>
<br/>
<br/>

<p align="center">
System Diagram<br/>
<img src="https://github.com/user-attachments/assets/5fdb2925-93d0-41e9-804b-533e94b03991" height="60%" width="60%" alt="System Diagram"/>
<br />


<h2>Languages and Utilities Used</h2>

- <b>PowerShell</b> 

<h2>Environments Used </h2>

- <b>Windows 11</b>

<h2>Program walk-through:</h2>

<h4>Path A: Collect a New Baseline</h4>

- calculates the current hash values for the target files and stores them in a file called <i>baseline.txt</i>


<p align="center">
Hash values for the target files <br/>
<img src="https://github.com/user-attachments/assets/a7d9387f-1d95-4ec0-a256-f17688f3ccd1" height="60%" width="60%" alt="Disk Sanitization Steps" />
<br/>


<h4>Path B: Monitor Files Using the Saved Baseline</h4>


<p align="center">
Monitoring Files<br/>
<img src="https://github.com/user-attachments/assets/36aafa2e-661f-45cc-ab0e-bc51e11d2001" height="60%" width="60%" alt="Disk Sanitization Steps" />
<br/>
 
