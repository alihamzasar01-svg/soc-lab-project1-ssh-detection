# SOC Home Lab, Project 1: Detecting Failed SSH Logins with Wazuh

I simulated failed SSH logins against a non-existent user on a Kali Linux machine and watched Wazuh detect them. This was my first hands-on SOC lab. It is small on purpose: one machine, one attack pattern, and a focus on understanding exactly what the SIEM shows and why.

> **Scope note:** Everything here ran inside my own lab, against my own machine (`localhost`). Nothing was tested against systems I don't own. Please do the same if you try it.

---

## Table of contents

1. [Goal](#goal)
2. [Lab environment](#lab-environment)
3. [What I did](#what-i-did)
4. [What Wazuh showed](#what-wazuh-showed)
5. [How to read the results](#how-to-read-the-results)
6. [A discrepancy I noticed](#a-discrepancy-i-noticed)
7. [Limitations](#limitations)
8. [What I learned](#what-i-learned)
9. [Next steps](#next-steps)
10. [Repository contents](#repository-contents)

---

## Goal

Answer one simple question: **if someone repeatedly fails to log in over SSH, what does that look like in a SIEM, and can I find it quickly?**

This is the basic loop of SOC work: generate an event, find it, understand it, and describe it clearly.

## Lab environment

| Item | Detail |
|---|---|
| Operating system | Kali Linux (virtual machine) |
| SIEM / XDR | Wazuh (manager and dashboard running on the same Kali machine, shown as `manager.name: kali`) |
| Target service | OpenSSH server on `localhost`, port 22 |
| Wazuh version | `v4.14.9` |
| Date of test | 6 November 2026 |

Because the "attacker", the target and the SIEM are all on one machine, the traffic never leaves the loopback interface. That is why the screenshots show `localhost` and `::1`, and no external IP addresses.

## What I did

### 1. Generated failed logins

From a terminal on Kali, I tried to SSH in as a user that does not exist and typed wrong passwords each time:

```bash
ssh invaliduser@localhost
```

SSH asked for a password three times. All three were wrong, and the connection closed:

```
invaliduser@localhost's password:
Permission denied, please try again.
invaliduser@localhost's password:
Permission denied, please try again.
invaliduser@localhost's password:
Connection closed by ::1 port 22
```

![Failed SSH logins in the Kali terminal](images/01_ssh_failed_logins.png)

### 2. Opened the Wazuh Threat Hunting dashboard

In the Wazuh dashboard I opened **Threat Hunting** with the time range set to **Last 24 hours** and a filter on `manager.name: kali`.

![Wazuh Threat Hunting overview](images/02_wazuh_overview.png)

The overview showed 118 total events, 0 alerts at level 12 or above, 5 authentication failures and 29 authentication successes. The MITRE ATT&CK panel listed Sudo and Sudo Caching, Valid Accounts, Password Guessing and SSH.

### 3. Filtered for authentication failures

I searched for the text `authentication failure` and the dashboard narrowed down to the failed login alerts. Wazuh mapped them to the MITRE ATT&CK technique **Password Guessing**, and the alert level chart showed the alerts at **level 5**.

![Filtered view of authentication failures](images/03_auth_failure_detected.png)

### 4. Compared with normal activity

To see what "normal" looks like, I ran this DQL query:

```
rule.groups: "authentication_success"
```

It returned 29 successful authentication events, mapped to **Valid Accounts**, with no failures in that view.

![Authentication successes for comparison](images/04_failure_vs_success.png)

## What Wazuh showed

| Observation | Where I saw it |
|---|---|
| Failed SSH attempts were turned into authentication failure alerts | Threat Hunting dashboard |
| Alerts were mapped to MITRE ATT&CK **Password Guessing** | MITRE ATT&CK panel |
| Alerts appeared at **level 5** | Alert level evolution chart |
| Normal logins looked different: **Valid Accounts**, level 3 | Success query |
| No high-severity alerts (level 12 or above) | Overview card |

## How to read the results

A single failed login means very little. People mistype passwords all the time. What makes failed logins interesting is the pattern:

- **Many failures in a short time** from one source suggests password guessing.
- **Failures against usernames that don't exist** (like `invaliduser`) suggest someone is guessing, not a real user who forgot a password.
- **A success right after a run of failures** is the worrying one, because it can mean the guessing worked.

In this lab I only produced the first two patterns, and only a few times. A real investigation would also ask where the source IP is from, how many attempts there were over what period, and whether any login succeeded afterwards.

## A discrepancy I noticed

The overview card showed **5** authentication failures, but my text search for `authentication failure` showed **2**.

My current understanding is that the overview card counts events by Wazuh's built-in rule group, while my search matched on the text of the alert, so the two can include different events. I have not fully confirmed this. I plan to open the **Events** tab, expand each alert, and compare `rule.id` and `rule.description` to see exactly which events each number includes.

I'm leaving this here instead of hiding it, because noticing a count that doesn't match is part of the job.

## Limitations

- **Single machine.** There is no real network, no separate attacker, and no external source IP.
- **Small sample.** A few failed attempts is not a brute-force attack. It is a test that proves the detection pipeline works.
- **No tuning.** I used Wazuh's default rules and did not write custom ones.
- **No response step.** I detected the activity but did not block it, for example with Fail2ban or firewall rules.
- **Dashboard only.** I worked mostly from the dashboard and have not yet analyzed the raw logs in depth.

## What I learned

- Failed and successful logins look clearly different in a SIEM, and a baseline of normal activity makes the failures easier to spot.
- Different dashboard elements can show different counts, because they filter differently. I need to check what each number actually counts before trusting it.
- Wazuh adds MITRE ATT&CK context automatically, which gives a useful first label for an alert, though it is not a verdict.
- Writing down what I did and what I'm unsure about taught me more than just running the commands.

## Next steps

- [ ] Resolve the 5 vs 2 discrepancy using the Events tab and the raw alert fields
- [ ] Repeat the test from a separate attacker VM so the source IP is real
- [ ] Run a larger attempt with a tool such as Hydra against my own lab host and see how alert levels change
- [ ] Add Fail2ban to block repeated failures
- [ ] Move on to Project 2: Windows event log monitoring with Sysmon

## Repository contents

```
.
├── README.md
└── images/
    ├── 01_ssh_failed_logins.png
    ├── 02_wazuh_overview.png
    ├── 03_auth_failure_detected.png
    └── 04_failure_vs_success.png
```

---

## About me

I'm building toward a SOC Analyst role, with a background in technical support and network operations. I'm documenting my labs as I go, mistakes included. Feedback is welcome, so feel free to open an issue or message me on LinkedIn: www.linkedin.com/in/alihamzasar001.
