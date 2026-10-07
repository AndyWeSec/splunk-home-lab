# splunk-home-lab
I built my own Splunk Enterprise SIEM at home by running the official Splunk container in Docker on an Apple Silicon Mac, then loaded the public Boss of the SOC v3 (BOTS v3) dataset into it so I could practise real investigations on realistic data. 
# 🔎 Splunk Home Lab: BOTS v3 Investigation on Docker

📅 **7 October 2026**
**Tools:** Splunk Enterprise, Docker Desktop, BOTS v3 dataset, AWS CloudTrail logs, MITRE ATT&CK

## 📋 Overview
I built my own Splunk Enterprise SIEM at home by running the official Splunk container in Docker on an Apple Silicon Mac, then loaded the public Boss of the SOC v3 (BOTS v3) dataset into it so I could practise real investigations on realistic data. This write-up documents the whole process: setting up the platform, loading the data, fixing the problems I hit along the way, and running my first investigation searches against the AWS CloudTrail logs.

The BOTS v3 dataset is a free, pre-indexed, attack-simulation dataset published by Splunk. It contains almost two million events from 107 different log sources (endpoint, network, firewall, cloud and web), which makes it a good sandbox for building threat-hunting and alert-triage skills without touching a real company's data.

## 🎯 Why this matters to a SOC analyst
- **Platform ownership:** Anyone can click through a hosted training room. Standing up the SIEM myself, loading data and debugging it shows I understand how a SIEM actually works underneath: where data lives, how indexes and apps are installed, and what permissions Splunk needs.
- **Troubleshooting under pressure:** Real SOC work involves broken log pipelines. I diagnosed a failed archive extraction by inspecting the file itself and fixed a file-ownership problem that stopped Splunk reading the dataset.
- **Investigation workflow:** I followed a repeatable approach: see what data I have, narrow to the log source that matters (CloudTrail), find which identities are active, research unfamiliar API calls, and map the behaviour to MITRE ATT&CK.
- **Interview answer:** To explain how I'd start hunting in an unfamiliar environment, I can walk through this lab: baseline the sourcetypes, pivot into one source, group activity by identity (`userIdentity.arn`), and look up anything I don't recognise.

## 🖥️ Lab environment

| Component | Detail |
|---|---|
| Host | Apple Silicon Mac (arm64) |
| Container platform | Docker Desktop, aarch64 Linux VM with 10 CPUs and 15.6 GiB memory |
| SIEM | Splunk Enterprise (`splunk/splunk:latest` image), web interface on port 8000 |
| Dataset | BOTS v3, about 319 MB compressed, loaded as the `botsv3` index |
| Data period | August 2018 (the bucket names in the dataset are epoch timestamps from that period) |

## 🛠️ Build walkthrough

### Step 1: Check the host and Docker
Before pulling anything, I checked the machine I was running on. `uname -m` returned `arm64` and `docker info` confirmed Docker Desktop was running an `aarch64` Linux VM. This matters because the Splunk image is built for Intel/AMD (`amd64`) processors, so on an Apple Silicon Mac it has to run through emulation.

<img width="1434" height="751" alt="1" src="https://github.com/user-attachments/assets/5927441a-a228-43e6-b0e5-0bc1dbf461e5" />

<img width="1434" height="751" alt="2" src="https://github.com/user-attachments/assets/f7988972-755c-47a4-97b5-95bae3ebfc3f" />

### Step 2: Launch the Splunk container
I started Splunk with a single command (password redacted here):

```bash
docker run -d --platform linux/amd64 -p 8000:8000 \
  -e "SPLUNK_START_ARGS=--accept-license" \
  -e "SPLUNK_GENERAL_TERMS=--accept-sgt-current-at-splunk-com" \
  -e "SPLUNK_PASSWORD=<strong-password>" \
  --name splunk splunk/splunk:latest
```

- `-d` runs the container in the background.
- `--platform linux/amd64` forces the Intel build so it runs under emulation on the ARM Mac.
- `-p 8000:8000` publishes Splunk's web interface on port 8000 of my machine.
- The `SPLUNK_START_ARGS` and `SPLUNK_GENERAL_TERMS` variables accept Splunk's licence and terms so the container can start unattended.
- `SPLUNK_PASSWORD` sets the `admin` password.
- `--name splunk` gives the container a fixed name so later commands can target it (`docker exec ... splunk`).

<img width="1470" height="788" alt="3" src="https://github.com/user-attachments/assets/214cbb9e-628d-41d3-ad4c-7e3d2defe514" />

### Step 3: Download the BOTS v3 dataset
I downloaded the dataset from Splunk's public S3 bucket with `curl -L -o botsv3_data_set.tgz <dataset URL>`. The `-L` flag follows redirects and `-o` saves to a named file. The archive is about 319 MB and took roughly 13 minutes at the speed I had.

<img width="1442" height="788" alt="7" src="https://github.com/user-attachments/assets/1102fffe-a2b1-46a7-9463-6009440a3e39" />

### Step 4: Extract the dataset into Splunk
With the archive in the container's `/tmp` folder, I extracted it as the root user straight into Splunk's apps directory:

```bash
docker exec -u root splunk tar -xzvf /tmp/botsv3_data_set.tgz -C /opt/splunk/etc/apps/
```

- `docker exec -u root splunk` runs a command inside the running container as root.
- `tar -xzvf` extracts (`x`) a gzip (`z`) archive and lists each file as it goes (`v`).
- `-C /opt/splunk/etc/apps/` extracts into the folder where Splunk looks for apps.

The listing shows how the dataset is packaged. It arrives as a Splunk app called `botsv3_data_set`, containing configuration files (`app.conf`, `indexes.conf`, `props.conf`, `transforms.conf`, `tags.conf`), a `lookups` folder, and the pre-indexed data itself under `var/lib/splunk/botsv3/db/`. Because the data is already indexed into Splunk's bucket format, I didn't need to ingest anything. The lookups included CSV files such as `eventcode.csv`, `scanner_agents.csv`, `ransomware_extensions.csv`, `ddns_provider.csv` and `iis_action_lookup.csv`, which help enrich searches (for example flagging ransomware file extensions or dynamic DNS providers).

<img width="914" height="439" alt="11" src="https://github.com/user-attachments/assets/d8878e8f-2460-408c-b355-e1fad9136af4" />

<img width="1442" height="819" alt="8" src="https://github.com/user-attachments/assets/f8e12171-2cb4-4e51-8a61-ebe9f208faad" />

<img width="914" height="439" alt="12" src="https://github.com/user-attachments/assets/0244eb95-5773-4e9f-ba7c-aca508d9661d" />

### Step 5: Fix ownership and restart Splunk
Because I extracted as root, the files belonged to root, and Splunk runs as the `splunk` user and couldn't read them properly. I fixed the ownership recursively and restarted the container so Splunk would load the new app and index:

```bash
docker exec -u root splunk chown -R splunk:splunk /opt/splunk/etc/apps/
docker restart splunk
docker ps
```

`docker ps` first showed the status as `health: starting`, and after about a minute it changed to `healthy`, which is when Splunk's web interface was ready.

<img width="1466" height="525" alt="10" src="https://github.com/user-attachments/assets/0cf5fde6-cbcd-424d-84da-d0ab47e2e76c" />

<img width="914" height="439" alt="13" src="https://github.com/user-attachments/assets/86b8200f-78f2-4e76-9af4-8bc5f7f7a674" />

### Step 6: Log in
I opened `localhost:8000` in the browser, signed in as `admin`, and landed on the Splunk Enterprise home page with Search & Reporting, Audit Trail and Data Management apps available.

<img width="1442" height="788" alt="4" src="https://github.com/user-attachments/assets/d7d11b3e-6e67-4ad8-95dc-3a56e4731888" />

<img width="1442" height="788" alt="5" src="https://github.com/user-attachments/assets/592380af-af91-4eef-89b1-e107c55e554c" />

## 🔧 Troubleshooting log
Two things went wrong. Both are the kind of problem a SOC analyst meets when a log pipeline isn't behaving.

### Problem 1: The archive wouldn't extract
- **Symptom:** `tar` stopped with an unrecoverable error instead of extracting the dataset.
- **Diagnosis:** I looked at the start of the file with `docker exec -u root splunk head -n 5 /tmp/botsv3_data_set.tgz`. Instead of the binary gibberish a gzip file starts with, the output was an HTML web page from AWS. So the "archive" in the container was not an archive at all. The download had saved a web page (for example an error or redirect page) under the `.tgz` name.
- **Fix:** Download the file again with `curl -L`, check that it is the full size (about 319 MB), and re-run the extraction.
- **Lesson:** When a file won't open, check what it really is before assuming the tool is broken. `head`, `file` and the file size are quick ways to verify.
- 
<img width="1466" height="746" alt="9" src="https://github.com/user-attachments/assets/01ff736f-c2bc-4a83-96b0-0ed73ec2a8af" />

### Problem 2: A typo in the ownership command
- **Symptom:** `Error response from daemon: No such container: chown`.
- **Diagnosis:** I had left out the container name, so Docker read `chown` as the name of the container to run the command in.
- **Fix:** Re-run it in the correct form, `docker exec -u root splunk chown -R splunk:splunk /opt/splunk/etc/apps/`.
- **Lesson:** With `docker exec`, the container name always comes before the command.

(The error and the corrected command are both visible in the Step 5 screenshot, `images/10.png`.)

## 🔎 First investigation: exploring the data

### Search 1: What data do I have?

```spl
index=botsv3 earliest=0 | stats count by sourcetype | sort -count
```

- `index=botsv3` limits the search to the dataset I loaded.
- `earliest=0` removes the start of the time window. This is needed because the events date from 2018, and Splunk's default time range would show nothing.
- `stats count by sourcetype` counts events for each type of log, and `sort -count` puts the biggest first.

The search returned **1,944,092 events across 107 sourcetypes**. This is the first step of any hunt in an unfamiliar environment: understand what you can see before you start looking for bad things.

| Sourcetype | Events | What it is |
|---|---|---|
| syslog | 283,976 | Standard system and network device logging |
| stream:ip | 227,872 | Network traffic metadata captured by Splunk Stream (IP level) |
| osquery:results | 219,997 | Results of osquery endpoint queries |
| stream:dns | 218,456 | DNS lookups seen on the network |
| stream:udp | 157,960 | UDP network traffic metadata |
| winhostmon | 129,679 | Windows host monitoring data |
| aws:cloudwatchlogs | 115,145 | AWS CloudWatch application and system logs |
| aws:cloudwatchlogs:vpcflow | 97,448 | AWS VPC flow logs (network connections inside AWS) |
| stream:tcp | 84,031 | TCP network traffic metadata |
| osquery:info | 83,961 | osquery status information |
| cisco:asa | 80,192 | Cisco ASA firewall logs |

<img width="1433" height="666" alt="14" src="https://github.com/user-attachments/assets/35a35f38-d557-49a1-9e30-47598d845dcf" />

### Search 2: Pivot into AWS CloudTrail
I chose one source to dig into: AWS CloudTrail, the audit log of every API call made in an AWS account.

```spl
index=botsv3 sourcetype=aws:cloudtrail earliest=0 | stats count by eventName | sort -count
```

This returned **6,571 events across 113 different API actions**. The most common were:

| eventName | Events | What it means |
|---|---|---|
| DescribeConfigRuleEvaluationStatus | 1,798 | AWS Config compliance checks, usually routine automation |
| RunInstances | 582 | Launching new EC2 virtual machines |
| DescribeInstances | 346 | Listing EC2 instances |
| AssumeRole | 332 | Taking on an IAM role and receiving temporary credentials |
| GetCallerIdentity | 304 | Asking AWS "who am I?" for the credentials in use |
| Decrypt | 292 | Decrypting data with AWS KMS keys |
| DescribeInstanceStatus | 221 | Checking the health of EC2 instances |
| DescribeConfigRules | 173 | Listing AWS Config rules |
| GetBucketLocation | 169 | Reading the region of an S3 bucket |
| GetBucketCors | 151 | Reading an S3 bucket's cross-origin settings |
| GetBucketLifecycle | 147 | Reading an S3 bucket's lifecycle rules |

The list is mostly read-only "Describe" and "Get" calls, which is typical background noise from monitoring and configuration tools. The ones worth a closer look in a real investigation are the identity and credential calls (`AssumeRole`, `GetCallerIdentity`) and anything that creates resources (`RunInstances`).

<img width="1433" height="666" alt="16" src="https://github.com/user-attachments/assets/429868a2-75b3-4e04-851b-f74439e2239d" />

<img width="1433" height="666" alt="17" src="https://github.com/user-attachments/assets/a17e8867-1c39-485e-bc65-ab06bd27f9dc" />

### Research: What does GetCallerIdentity do?
I didn't recognise every event name, so I checked the official AWS Security Token Service documentation. `GetCallerIdentity` returns details about the IAM user or role whose credentials were used to make the call. The notable point is that **it needs no permissions at all**. Even if an administrator explicitly denies the action in a policy, the call still works, because the same information is returned when access is denied.

Why that matters for defenders: attackers who steal AWS keys commonly run `GetCallerIdentity` first to find out whose credentials they hold and whether the keys still work, and the call can't be blocked. That does not make every `GetCallerIdentity` event malicious, as SDKs and automation use it routinely. The analyst's job is to baseline which identities normally call it and to investigate the unfamiliar ones.

<img width="1433" height="666" alt="18" src="https://github.com/user-attachments/assets/03b5a957-4d65-4459-8bf1-735cedda17ba" />

### Search 3: Who is making these calls?

```spl
index=botsv3 sourcetype=aws:cloudtrail earliest=0 | stats count values(eventName) as actions by userIdentity.arn | sort -count
```

- `userIdentity.arn` is the Amazon Resource Name of the identity behind each API call.
- `values(eventName) as actions` collects the distinct API calls each identity made into one list.

There were **17 distinct identities**. The busiest was the IAM user `splunk_access`, with **4,091 of the 6,571 events (about 62%)**, performing a wide range of `Describe...` and `Get...` calls. A high-volume IAM user that only reads configuration looks like an automated collection account, which would be normal. The next step would be to compare it with the other 16 identities and look for any that behave differently, such as creating resources, assuming roles unexpectedly, or appearing at unusual times.

<img width="1433" height="666" alt="19" src="https://github.com/user-attachments/assets/88e296b9-a034-43eb-bf58-df394ed9846d" />

## 🗺️ Mapping to MITRE ATT&CK
Because the CloudTrail data revolves around credentials and identities, I reviewed the MITRE ATT&CK technique for abusing them: **Valid Accounts (T1078)**.

- **What it is:** Adversaries obtain and abuse credentials of existing accounts to gain access, stay in the environment, escalate privileges or avoid detection. Using legitimate credentials can also avoid the need for malware at all, which makes it harder to spot.
- **Tactics it covers:** Stealth, Persistence, Privilege Escalation and Initial Access.
- **Sub-techniques:** Default Accounts (T1078.001), Domain Accounts (T1078.002), Local Accounts (T1078.003) and Cloud Accounts (T1078.004).
- **Relevance here:** The platforms for this technique include IaaS (cloud infrastructure). Stolen or misused AWS IAM credentials seen in CloudTrail would be tracked under **Cloud Accounts (T1078.004)**, so identity-focused searches like the one above are how an analyst would hunt for it.

<img width="1433" height="666" alt="15" src="https://github.com/user-attachments/assets/bdea3443-5ecf-4197-8f45-c895ff3ba168" />

## 🧠 Skills demonstrated
- Deploying and configuring Splunk Enterprise in a Docker container, including handling CPU architecture differences
- Loading and verifying a large third-party dataset into a SIEM
- Linux file ownership and permissions, and basic Docker command-line troubleshooting
- Verifying file integrity and diagnosing failed downloads
- Writing SPL searches using `stats`, `values`, `sort` and time modifiers
- Baselining log sources and identifying the identities behind cloud activity
- Researching unfamiliar API events in vendor documentation
- Mapping observed behaviour to MITRE ATT&CK

## 🔒 Lab hardening and next steps
- **Restrict network exposure:** My container published port 8000 on all network interfaces. Binding it to localhost only (`-p 127.0.0.1:8000:8000`) stops other devices on the network reaching the lab.
- **Use a strong password:** A simple lab password is fine for a throwaway test, but because the interface was reachable on the network, a strong, unique password is the safer habit. Never publish it in write-ups.
- **Pin the image version:** Using `splunk/splunk:latest` means a rebuild could behave differently later. Pinning a version keeps the lab reproducible.
- **Next investigations:** Compare `splunk_access` with the other 16 identities, hunt `AssumeRole` and `RunInstances` activity, and pivot to the network and endpoint sourcetypes (`stream:dns`, `cisco:asa`, `osquery:results`) to build a full attack timeline.

---
*Dataset: Boss of the SOC v3, published by Splunk. All data is simulated.*
