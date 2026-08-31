# Case 001: Repeated Failed RDP Logon Attempts (Simulated)

**Environment:** SOC Home Lab (pfSense / AD_LAB subnet / Windows 10 VM / Kali Linux)
**Detection Source:** Microsoft Sentinel (Azure Arc + AMA Data Collection Rule)
**Event Type:** Windows Security Event ID 4625 (An account failed to log on)
**Classification:** Simulated / Test Activity

---
## Indicators Observed

| Field           | Value                                                         |
| --------------- | ------------------------------------------------------------- |
| Event ID        | 4625 (x5)                                                     |
| Time Range      | 8/30/2026, 7:34:25.351 AM UTC – 8/30/2026, 7:34:25.970 AM UTC |
| Source IP       | 10.0.0.10                                                     |
| Target Account  | \test                                                         |
| Target Computer | Win10-VM1.ad.lab                                              |
| Logon Type      | 3 (Network)                                                   |
| SubStatus       | 0xC000006A                                                    |
**KQL Query Used:**
``` kql
SecurityEvent
| where TimeGenerated > ago(24h) and EventID == 4625
| project TimeGenerated, Account, Computer, IpAddress, LogonType, SubStatus
| sort by TimeGenerated asc
```

![](attachments/Sentinel%20query%20results.png)

---
## Analysis

The attack speed was fast, with 5 attempts occurring in under a second, which can indicate an automated attack/automation tool was used rather than a human typing credentials manually.

There was no domain field populated for the account (`\test`). This is likely due to the tooling used for the login attempt (xfreedrdp was invoked without specifying a domain). While this indicates the login attempt didn't specify a domain scope, it may or may not reflect the attacker's actual knowledge of the domain structure. The account is confirmed to exist within Active Directory, and the blank domain field should not be treated as conclusive evidence about attacker awareness.

Account validity was confirmed by SubStatus 0xC000006A, which indicates the target account is real but the password used was incorrect. This raises the urgency level, as this account is now a confirmed, known target — as opposed to an attempt against a nonexistent account (which would return SubStatus 0xC0000064 and generally represent a lower-priority signal). 

---
## Verdict
**Classification: Simulated / Test Activity**
This is a simulated exercise; there is no real threat, as the source of the attack, the target account, and the tooling used were all fully controlled by the analyst. The exercise demonstrates a built and working detection pipeline that correctly captures, ingests, and attributes events of this type — confirming that Sentinel, the AMA Data Collection Rule, and the underlying KQL query are functioning as intended for future case work.

---
## Recommended Runbook
**1. Immediate Triage**
- Check the targeted account and review its group memberships/permissions to assess potential impact if compromised.
- Search for the source IP across all accounts (not just the one targeted) to determine whether this is a focused brute-force attempt (one account, many passwords) or a password-spraying pattern (one/few passwords, many accounts).

**2. Containment / Response**
- Consider locking the targeted account and blocking the source IP, weighed against the risk of disrupting a legitimate user (avoid automatic lockout purely on this signal without corroboration).
- Corroborate with additional indicators before escalating to full containment — e.g., geographic origin of the source IP (login attempts from an unexpected country), time-of-day anomalies, or whether the account owner was expected to be authenticating at all.

**3. Preventive / Policy**
- Implement an account lockout policy after a defined number of failed attempts.
- Scale the lockout threshold based on account sensitivity — e.g., stricter thresholds for accounts with access to sensitive systems or data, more lenient thresholds for low-risk accounts.

---
## Notes / Limitations
- This case used a single account and a single repeated wrong password, and therefore demonstrates a "repeated failed login attempt" pattern specifically — not a full multi-password brute-force or multi-account password-spraying pattern. Those variations are planned for later cases.
- LogonType 3 (not 10) was observed because the failure occurred during Network Level Authentication (NLA/CredSSP), which is a network-layer credential check that happens before any interactive desktop session is established.