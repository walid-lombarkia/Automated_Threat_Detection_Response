# 🛡️ Automated Threat Detection & Response — SOC 
 
> A fully functional SOC automation pipeline built from scratch: endpoint telemetry → SIEM detection → automated triage → case management → analyst notification.
 
---
 
## 📌 Overview
 
This project replicates a real-world Security Operations Center (SOC) workflow. It detects credential-dumping attacks, enriches alerts with threat intelligence, automatically opens cases in a case management platform.
 
**Core stack:** Sysmon · Wazuh · Shuffle · TheHive · VirusTotal 
 
---
 
## 🗺️ Architecture
<img width="800" height="600" alt="421639955-5fdb2925-93d0-41e9-804b-533e94b03991" src="https://github.com/user-attachments/assets/ab6f8e74-4c50-4e5d-9100-44e457d215a7" />
 
---
 
## 🧰 Tools & Technologies
 
| Category | Tool | Purpose |
|---|---|---|
| Endpoint Logging | **Sysmon + SwiftOnSecurity config** | Deep Windows event telemetry (Process creation, network, file events) |
| SIEM | **Wazuh 4.10** | Log ingestion, threat detection, custom rules |
| SOAR | **Shuffle** | Workflow automation, hash extraction, API orchestration |
| Case Management | **TheHive 5** | Incident alerting and analyst assignment |
| Threat Intel | **VirusTotal API** | SHA-256 hash reputation lookup |
| Notification | **Telegram Bot** | Real-time analyst alerts |
| Log Backend | **Elasticsearch** | Log storage and indexing |
| Scripting | **PowerShell** | Agent deployment, Mimikatz download |
 
---
 
## 🔬 Environments
 
- **Windows 10** — Monitored endpoint running Wazuh agent + Sysmon
- **Ubuntu 22.04 LTS (VM)** — Wazuh Manager + TheHive + Shuffle
---
 
## 📋 Step-by-Step Walkthrough
 
### Step 1 — Install & Configure Sysmon for Deep Windows Event Logging
 
Sysmon provides granular Windows telemetry far beyond native Windows Event Logs. It captures process creation, network connections, file creation, and more — critical for detecting living-off-the-land attacks.
 
**What was done:**
1. Downloaded `Sysmon64.exe` and the [SwiftOnSecurity `sysmonconfig.xml`](https://github.com/SwiftOnSecurity/sysmon-config) ruleset
2. Installed Sysmon with the hardened config:
   ```powershell
   .\Sysmon64.exe -accepteula -i sysmonconfig.xml
   ```
3. Verified the service is running:
   ```powershell
   Get-Service Sysmon64
   ```
<img width="437" height="96" alt="image" src="https://github.com/user-attachments/assets/a52d9c89-d56e-4e72-829d-d5f2941dd0f2" />

   
4. Confirmed events are visible in **Event Viewer** under:
   `Applications and Services Logs > Microsoft > Windows > Sysmon > Operational`
<img width="700" height="400" alt="image" src="https://github.com/user-attachments/assets/b6618c8c-6f63-42de-9e31-df678a07090f" />

- PowerShell output showing Sysmon directory listing and `Get-Service Sysmon64` returning `Running`
- Event Viewer displaying Sysmon Operational log with Event ID 11 (File Create) entries and rule name `Services File Permissions Weakness`
---
 
### Step 2 — Set Up Wazuh & TheHive for Threat Detection & Case Management
 
#### 2a. Wazuh Installation
 
Wazuh is the SIEM backbone — it ingests Sysmon logs from the Windows endpoint, normalizes them, and applies detection rules.
 
**Wazuh Manager (on Ubuntu):**
```bash
curl -sO https://packages.wazuh.com/4.10/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```
Wazuh dashboard login screen at `192.168.222.134`
  <img width="1084" height="740" alt="image" src="https://github.com/user-attachments/assets/f768e26f-644e-4a1e-a87d-b89e375f0b12" />
  
**Wazuh Agent (on Windows endpoint, via PowerShell):**
```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.10.4-1.msi `
  -OutFile $env:tmp\wazuh-agent
msiexec.exe /i $env:tmp\wazuh-agent /q `
  WAZUH_MANAGER='192.168.222.134' `
  WAZUH_AGENT_NAME='Waylid'
net.exe start WazuhSvc
```
PowerShell showing `WazuhSvc` starting successfully
  <img width="1008" height="149" alt="image" src="https://github.com/user-attachments/assets/57c86fea-5a81-4e1e-b041-9bb8de81a651" />
  
**Configure Wazuh agent to forward Sysmon logs** by editing `ossec.conf`:
```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```
 
Then restart the agent:
```powershell
Restart-Service Wazuh
```
Wazuh overview dashboard showing agent summary and 24-hour alert counts (62 medium severity, 19 low)
  <img width="1271" height="752" alt="image" src="https://github.com/user-attachments/assets/6ecca676-9edf-4f6b-b62e-aad6f95debfb" />


Wazuh Discover page filtered on `sysmon` showing **267 hits** of Sysmon events from agent `Waylid`
  <img width="1636" height="885" alt="image" src="https://github.com/user-attachments/assets/d60b361c-c954-4d7a-921b-b176c3312b24" />


#### 2b. TheHive Installation
 
TheHive is the case management platform. SOC analysts receive incidents here, track investigations, and coordinate response.

Thehive installation:
```bash
wget -q -O /tmp/install_script.sh \
  https://scripts.download.strangebee.com/latest/sh/install_script.sh
sudo -v
bash /tmp/install_script.sh
```
 
**Post-install configuration:**
 
TheHive depends on **Cassandra** (storage) and **Elasticsearch** (indexing). Both must be configured to listen on the TheHive server's IP rather than localhost.
 
Key config files edited:
- `/etc/cassandra/cassandra.yaml` — set `listen_address`, `rpc_address`, and `seeds` to `192.168.222.135`
  
    <img width="600" height="400" alt="listen_interface" src="https://github.com/user-attachments/assets/867eb076-00c0-4a4b-9d81-6fe86ee729b8" /></br>
    <img width="600" height="71" alt="rpc_address" src="https://github.com/user-attachments/assets/33004c44-efe4-4003-9838-3ed883da9c15" />

- `/etc/elasticsearch/elasticsearch.yml` — set `http.host`, `cluster.name: thehive`, `node.name: thehive-node`
    <img width="889" height="373" alt="image" src="https://github.com/user-attachments/assets/8002893d-94b5-4312-bb08-610945b821af" /> 
  
- `/etc/thehive/application.conf` — updated Cassandra and Elasticsearch hostnames to the server IP
    <img width="897" height="599" alt="image" src="https://github.com/user-attachments/assets/7f05e9d0-89b7-4672-89e5-46f99cd62c9a" />

Fix directory ownership:
```bash
chown -R thehive:thehive /opt/thp
```
Terminal output confirming `thehive:thehive` ownership
   <img width="897" height="599" alt="image" src="https://github.com/user-attachments/assets/636939c8-4c46-4a72-9dd6-32d902f30a36" />

Verify services are listening:
```bash
sudo ss -tulpn | grep -E '9200|9042'
```


TheHive Organisation List in browser showing `admin` org created by TheHive system user
  <img width="1651" height="747" alt="image" src="https://github.com/user-attachments/assets/e7e03d5e-60ac-4267-889a-9a61c7025e59" />

---
 
### Step 3 — Execute Mimikatz & Create Detection Rules in Wazuh
 
Mimikatz is a credential-dumping tool commonly used by attackers post-compromise. This step validates that the detection pipeline works end-to-end.
 
#### 3a. Prepare the Endpoint
 
Added `C:\Users\Win10\Downloads` as a Windows Defender exclusion to allow Mimikatz to execute for lab purposes (simulating an attacker bypassing AV).
 <img width="1021" height="540" alt="image" src="https://github.com/user-attachments/assets/7629ed9e-0ae7-4b57-b885-3f0f2051fa35" />

#### 3b. Download Mimikatz
 
```powershell
$url = "https://github.com/gentilkiwi/mimikatz/releases/download/2.2.0-20220919/mimikatz_trunk.zip"
$output = "C:\Users\Win10\Downloads\mimikatz.zip"
Invoke-WebRequest -Uri $url -OutFile $output
Unblock-File -Path $output
```
 
#### 3c. Enable Full Log Archiving in Wazuh
 
On the Wazuh manager, edit `/var/ossec/etc/ossec.conf`:
```xml
<logall>yes</logall>
<logall_json>yes</logall_json>
```
 <img width="646" height="478" alt="image" src="https://github.com/user-attachments/assets/9780268f-6dc1-496b-bef4-9380b650cf3b" />

Enable archives in Filebeat (`/etc/filebeat/filebeat.yml`):
```yaml
filebeat.modules:
  - module: wazuh
    alerts:
      enabled: true
    archives:
      enabled: true
```
 <img width="646" height="381" alt="image" src="https://github.com/user-attachments/assets/7125054e-e00f-41e0-a707-286fa3cd91bc" />

Create an index pattern `wazuh-archives-*` in the Wazuh dashboard (Kibana) to search archived logs.
 <img width="1651" height="822" alt="image" src="https://github.com/user-attachments/assets/387960d9-fef2-4d02-b58a-20f25300beb7" />

#### 3d. Create a Custom Mimikatz Detection Rule
 
In Wazuh: **Server Management → Rules → Manage rules files → 0800-sysmon_id_1.xml**
 
Added a custom rule that detects Mimikatz by its **original filename in PE metadata** — meaning it fires even if the attacker renames the binary:
 
```xml
<rule id="100002" level="10">
  <if_group>sysmon_event1</if_group>
  <field name="win.eventdata.originalFileName" type="pcre2">(?i)mimikatz\.exe</field>
  <description>Mimikatz Usage Detected</description>
  <mitre>
    <id>T1003</id>
  </mitre>
</rule>
```
 <img width="1636" height="787" alt="image" title="Windows Defender Exclusions settings showing `C:\Users\Win10\Downloads` added" src="https://github.com/user-attachments/assets/0a5ed590-9320-4cbc-9318-76311dc2e62f" />

> **Why `originalFileName` matters:** Even if `mimikatz.exe` is renamed to `mimisafe.exe`, the PE header still contains the original filename. This was verified live — renaming the binary and re-running it still triggered Rule 100002.
 <img width="731" height="171" alt="image" src="https://github.com/user-attachments/assets/dce2fc5e-0bdc-45d1-9ef1-286a98f110cc" />


Alert document detail with `data.win.eventdata.originalFileName: mimikatz.exe` visible even though the executed binary was `mimisafe.exe`
  <img width="1651" height="790" alt="image" src="https://github.com/user-attachments/assets/5e147bac-5182-4b61-8809-8dbb0d05afc1" />
  <img width="822" height="740" alt="image" src="https://github.com/user-attachments/assets/5e0274a4-f01a-40ba-add7-7915627f02fe" />

---
 
### Step 4 — Automate Everything with Shuffle (SOAR)
 
Shuffle is the automation layer that connects Wazuh alerts to enrichment and response actions.
 
#### 4a. Install Shuffle
 
```bash
sudo apt update && sudo apt install -y docker.io docker-compose
sudo systemctl enable docker && sudo systemctl start docker
git clone https://github.com/Shuffle/Shuffle.git
cd Shuffle
sudo docker-compose up -d
```
 
Access at `http://<shuffle-ip>:3001` 
<img width="1255" height="854" alt="image" src="https://github.com/user-attachments/assets/6f6c98c4-bd19-42cb-ae43-7debab4bd8f2" />

 
#### 4b. Wazuh → Shuffle Webhook Integration
 
On the Wazuh manager, edit `/var/ossec/etc/ossec.conf` to add the Shuffle webhook:
 
```xml
<integration>
  <name>shuffle</name>
  <hook_url>http://192.168.204.151:3001/api/v1/hooks/webhook_82a81730-b6b5-476f-8eab-225c6b165fda</hook_url>
  <rule_id>100002</rule_id>
  <alert_format>json</alert_format>
</integration>
```
 <img width="648" height="577" alt="image" src="https://github.com/user-attachments/assets/cb66810d-b4c0-49fd-b0d9-dcd78e75a384" />

```bash
systemctl restart wazuh-manager
```

 <img width="1297" height="677" alt="image" src="https://github.com/user-attachments/assets/c4276b21-1b0e-4f80-a901-5732f7f3a5a2" />


This sends Mimikatz alerts (Rule ID 100002) directly to Shuffle as JSON payloads.
 
#### 4c. Shuffle Workflow — Full Pipeline
 
The automated workflow performs 5 actions in sequence:
 
**1. Wazuh Alerts** *(Webhook trigger)*
Receives the JSON alert from Wazuh containing process details, hashes, and event metadata.
 
**2. SHA256 Extraction** *(Regex capture group)*
Extracts the SHA-256 hash from `win.eventdata.hashes` using:
```
SHA256=([0-9A-Fa-f]{64})
```
 <img width="1427" height="797" alt="image" src="https://github.com/user-attachments/assets/9b314f4c-4c6b-4c32-a1a9-c278a6f3eceb" />
 <img width="1411" height="486" alt="image" src="https://github.com/user-attachments/assets/0d76efae-6201-4769-9010-895ba2f7a03b" />
 <img width="1705" height="988" alt="image" src="https://github.com/user-attachments/assets/f5eaa7f0-ed2d-41a8-aced-c6677f002e07" />
And here we obtained the file's hash to verify it on VirusTotal
 <img width="1583" height="912" alt="image" src="https://github.com/user-attachments/assets/9ef01b1a-35e9-4fa3-a24a-5b1ffcd540d0" />

**3. VirusTotal Hash Lookup** *(GET hash report)*
Submits the extracted hash to VirusTotal API and retrieves malicious/suspicious/harmless detection counts.

Input: `$sha256.group_0.#`
 <img width="1302" height="686" alt="image" src="https://github.com/user-attachments/assets/7ce34d6f-b162-4e0d-8535-8e6d6aa54f1b" />
 <img width="1827" height="922" alt="image" src="https://github.com/user-attachments/assets/bb4f6de0-f76a-485c-8809-676704f8ec49" />
 <img width="1875" height="1071" alt="image" src="https://github.com/user-attachments/assets/51d4477e-98e5-491c-9c22-a69405a1a61b" />


**4. TheHive Alert Creation** *(POST create alert)*
Forwards enriched incident data to TheHive:
```
Title:       $exec.title  (e.g. "Mimikatz Usage Detected")
Severity:    2 (Medium)
Summary:     Mimikatz Activity Detected on host: [hostname]
             Process ID: [pid] | Commandline: [cmd]
Tags:        ["T1003"]
Source:      Wazuh
Source Ref:  RuleId: 100002
Status:      New
Type:        Internal
```

Certain fields need to be completed.
```
Summary -> Mimikatz Activity Detected on host: $exec.text.win.system.computer and the Process Id: $exec.text.win.eventdata.processId and the Commandline: $exec.text.win.eventdata.commandLine
Description -> Mimikatz Detected on host:$exec.text.win.system.computer
```
<img width="235" height="457" alt="image" src="https://github.com/user-attachments/assets/c79948e6-de3d-4801-a21e-4d6ab2c9e099" />


 
#### TheHive — Org Setup
 
Created an organization `Dotcom` with two users:
<img width="1650" height="648" alt="image" src="https://github.com/user-attachments/assets/6fcc1cd9-48c5-41d6-9e1f-ab551f9472e5" />

- **SOAR** (service account for Shuffle integration — API key generated)
  <img width="1651" height="761" alt="image" src="https://github.com/user-attachments/assets/d37ff5fe-9448-4f23-ba09-d87f37126d66" />

- **waylid** (analyst account for case review)
  <img width="1642" height="802" alt="image" src="https://github.com/user-attachments/assets/125f561b-9f45-477b-8a02-b7cb79e7683b" />

- TheHive `Create alert` config with severity, summary, tags, and description fields
  <img width="1426" height="801" alt="image" src="https://github.com/user-attachments/assets/aba224a8-52fa-4004-bcc2-289731aae591" />

- Full Shuffle workflow with `Wazuh Alerts → SHA256 → Virustotal → TheHive` chain visible
  <img width="805" height="675" alt="image" src="https://github.com/user-attachments/assets/53bc2a43-6ad7-4024-911d-cb8b6236548b" />

- TheHive Alerts dashboard (logged in as `waylid`) showing "Mimikatz Usage Detected" alert: source Wazuh, RuleId 100002, tag T1003
  <img width="1650" height="575" alt="image" src="https://github.com/user-attachments/assets/a9dcd83c-544a-4129-8a59-7796d33b761d" />

- TheHive alert detail view showing full description, summary with host/commandline, and status `New`
  <img width="1651" height="760" alt="image" src="https://github.com/user-attachments/assets/97bbca0f-ae91-45d8-90da-51e620d3b3e3" />

---

**5. Telegram Notification** *(Send message to SOC analyst)*
Sends a real-time alert to the analyst's Telegram bot, prompting investigation.
#### 5a. Telegram Bot Setup

1. Open Telegram → search `@BotFather` → type `/newbot`
2. Follow instructions → save the bot token
3. Search `@userinfobot` → type `/start` → save your Chat ID
4. Open your bot in Telegram and send it any message first
   (required before it can send you messages)

#### 5b. Telegram node configuration in Shuffle

- App: Telegram Bot
- Action: Send message
- API token: ( BotFather token for our bot)
- Chat ID: (Chat ID from @userinfobot)
- Text:
  ```
   Mimikatz Activity Detected on host: $exec.text.win.system.computer
   Process Id: $exec.text.win.eventdata.processId
   Commandline: $exec.text.win.eventdata.commandLine
   Description -> Mimikatz Detected on host:$exec.text.win.system.computer
  ```
The completed workflow connects all components:

  <img width="700" height="550" alt="image" src="https://github.com/user-attachments/assets/c8da80fa-1cdc-47dd-be8a-c828602643fe" />

 And the result: </br>
  <img width="400" height="250" alt="image" src="https://github.com/user-attachments/assets/0ce4379a-d5bb-4e28-bdd5-8b00f6684e5a" />

## Phase 2: AI Alert Summarizer — The SOC Co-Pilot


The problem this solves: Alert fatigue is the #1 challenge in every SOC. Analysts waste minutes reading raw JSON just to answer "what is this alert telling me?" This phase plugs a local AI model directly into the existing Shuffle pipeline. By the time an analyst opens TheHive, the AI has already written a structured triage note automatically.


The VMware GPU Problem & Why We Run Ollama on the Host

The problem: If your Ubuntu Server runs inside VMware Workstation, your VM cannot access your physical GPU.

The solution: Keep all project code inside the Ubuntu VM. Let the host machine handle the heavy AI inference via network.

### 6 — Install Ollama 

```
irm https://ollama.com/install.ps1 | iex
```
<img width="462" height="120" alt="image" src="https://github.com/user-attachments/assets/bf59393c-1d2b-47e4-a5c8-9c294e473df3" />


### 7 — Expose Ollama to the VM Network
 
By default Ollama only accepts connections from `localhost`. You need to expose it over the VMware network.
 
**Set the environment variable on Windows:**
1. Open **Start** → search **"Environment Variables"**
2. Under System variables → **New**
3. Name: `OLLAMA_HOST` | Value: `0.0.0.0`
4. Run llama3.1
 ```
"%USERPROFILE%\AppData\Local\Programs\Ollama\ollama.exe" run llama3.1
```
 <img width="1302" height="252" alt="image" src="https://github.com/user-attachments/assets/36847649-318a-41e6-897d-a7fd848452af" />

**Verify connectivity from the Ubuntu VM:**
```bash
curl http://192.168.0.157:11434/api/generate -d '{
  "model": "llama3.1",
  "prompt": "Hello, are you online?",
  "stream": false
}'
```
 
Response:
```json
{"model":"llama3.1","response":"Yes, I am online!","done":true}
```
 
---
 
### 8 — Add the Ollama Node to Shuffle
 
Open your existing Shuffle workflow. After the **VirusTotal** node and before **TheHive**, add a new **HTTP node**.
 
Rename it to: `Ollama SOC Copilot`
<img width="852" height="467" alt="image" src="https://github.com/user-attachments/assets/c147edf9-0041-4f1b-8aa5-5e6bb5460b56" />


| Field | Value |
|---|---|
| Call Type | `POST` |
| URL | `http://192.168.0.157:11434/api/generate` |
| Content-Type | `application/json` |
 
**Body:**
```json
{
  "model": "llama3.1",
  "stream": false,
  "prompt": "You are a SOC analyst writing a triage note. Respond with EXACTLY 3 bullet points, no other text. CRITICAL: Replace all backslashes with forward slashes in any file paths.\n• WHAT HAPPENED: [the attack technique in one sentence]\n• KEY INDICATORS: [the executable name, SHA256 hash, and user]\n• RECOMMENDED ACTION: [what the analyst should do immediately]\n\nAlert data:\nHost: $exec.text.win.system.computer\nUser: $exec.text.win.eventdata.user\nProcess: $exec.text.win.eventdata.image\nOriginal name: $exec.text.win.eventdata.originalFileName\nCommand: $exec.text.win.eventdata.commandLine\nMITRE Technique: $exec.text.win.eventdata.ruleName\nSHA256: $sha256.group_0.0\nParent process: $exec.text.win.eventdata.parentImage\nIntegrity level: $exec.text.win.eventdata.integrityLevel"
}
``` 
---
 
### 9 — Update the TheHive Node
 
In your TheHive alert creation node, update the **Description** field:
 
```
AI TRIAGE Llama 3.1 via Ollama:/n
$ollama_soc_copilot.body.response
 
```
 
Update **sourceRef** to prevent duplicate alert errors across runs:
```
$exec.rule_id-$exec.id
```
 
---
 
 
### Result — AI-Enriched Alert in TheHive
 
When you open TheHive, every alert now shows:
 <img width="1136" height="717" alt="image" src="https://github.com/user-attachments/assets/adf67b0c-a8c1-4a95-8ed6-2408b5cf134c" />
 
---
 
### Variable Reference Map
 
| Data Point | Shuffle Variable |
|---|---|
| Alert title | `$exec.title` |
| Wazuh rule ID | `$exec.rule_id` |
| Unique event ID | `$exec.id` |
| Timestamp | `$exec.timestamp` |
| Hostname | `$exec.text.win.system.computer` |
| Malware image path | `$exec.text.win.eventdata.image` |
| Original filename | `$exec.text.win.eventdata.originalFileName` |
| Command line | `$exec.text.win.eventdata.commandLine` |
| Affected user | `$exec.text.win.eventdata.user` |
| MITRE technique | `$exec.text.win.eventdata.ruleName` |
| Integrity level | `$exec.text.win.eventdata.integrityLevel` |
| Parent process | `$exec.text.win.eventdata.parentImage` |
| SHA256 (extracted) | `$sha256.group_0.0` |
| VT file ID | `$virustotal.0.body.data.id` |
| **AI triage note** | **`$ollama_soc_copilot.body.response`** |
 
---
 
## 🔄 End-to-End Flow Summary
 
```
1.  Mimikatz executed on Windows endpoint (even if renamed to mimisafe.exe)
2.  Sysmon logs Event ID 1 (Process Create) — originalFileName = mimikatz.exe
3.  Wazuh agent forwards log → Wazuh Manager matches Rule 100002
4.  Wazuh fires webhook → Shuffle receives JSON alert
5.  Shuffle extracts SHA-256 hash via regex
6.  Shuffle queries VirusTotal → confirms malicious
7.  Shuffle sends alert data to Ollama (Llama 3.1 on host GPU)
8.  AI generates 3-bullet triage note in plain English
9.  Shuffle creates alert in TheHive — AI note at the top, raw context below
10. Telegram bot notifies SOC analyst in real time
11. Analyst opens TheHive — triage is already done
```
