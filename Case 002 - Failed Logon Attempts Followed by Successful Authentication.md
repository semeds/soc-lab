# Case 002: Failed Logon Attempts Followed by Successful Authentication

**Environment:** SOC Home Lab (pfSense / AD_LAB subnet / Windows 10 VM / Kali Linux)
**Detection Source:** Microsoft Sentinel (Azure Arc + AMA Data Collection Rule)
**Event Type:** Windows Security Event ID 4625 (An account failed to log on) and 4624 (An account was successfully logged on)
**Classification:** Simulated / Ambiguous Activity

---

## Indicators Observed

| Field | Value |
|---|---|
| Event ID | 4625 (x4), 4624 (x2) |
| Time Range | 8/31/2026, 6:55:14.398 AM UTC – 8/31/2026, 7:00:46.310 AM UTC |
| Source IP | 10.0.0.10 (consistent across all attempts) |
| Target Account | \test (failures) / AD\test (success) |
| Target Computer | Win10-VM1.ad.lab |
| Logon Type | 3 (Network) on all failures and on first success event; 10 (RemoteInteractive) on final success event |
| SubStatus | 0xC000006A (all failures) |
| Failure Cadence | ~30 seconds apart (4 attempts) |
| Gap Before Success | ~4 minutes after last failure |

**KQL Query Used:**

```kql
SecurityEvent
| where EventID in (4625, 4624)
| where Account in ("AD\\test", "\\test")
| where TimeGenerated between (datetime(2026-08-31T06:55:14.398) .. datetime(2026-08-31T07:00:46.310))
| project TimeGenerated, Account, Computer, IpAddress, EventID, LogonType, SubStatus
```

![](attachments/case%20002%20query%20results%201.png)
---

## Analysis

The failure cadence (~30 seconds between each of the 4 attempts) is consistent with a human manually retyping a password rather than an automated/scripted tool, which would typically show sub-second intervals (as seen in Case 001). This timing alone does not rule out a deliberate manual guessing attempt by an attacker, but it is more consistent with a legitimate user struggling to recall their password.

All four failures returned SubStatus 0xC000006A, confirming the account is valid and the password used was simply incorrect — not an attempt against a nonexistent account.

The successful logon produced two 4624 events one second apart. The first was still logged as LogonType 3, reflecting NLA (Network Level Authentication) validating the now-correct credentials at the network layer. The second was logged as LogonType 10, marking the point at which the interactive RDP session was actually established. This confirms that credential validation (authentication) and session establishment (authorization to use Remote Desktop) are distinct steps — a successful NLA check does not guarantee an interactive session will follow; the target account must also hold Remote Desktop Users (or equivalent) permissions on the host. This was confirmed directly during simulation: the `test` account initially authenticated successfully but failed to establish a session until it was added to the local Remote Desktop Users group.

The account field also changed between failures and success: failed attempts were logged as `\test` with no domain qualifier, while the successful attempt was logged as `AD\test`. A rejected NLA attempt never proceeds far enough for Windows to resolve which domain the account belongs to, so it is logged without qualification. A successful attempt completes domain validation, so the account is logged fully qualified. This distinction is a byproduct of how far authentication proceeded, not evidence of attacker knowledge or intent.

The source IP was identical (10.0.0.10) across every attempt, both failed and successful. In this lab topology, source IP cannot be used as a discriminating indicator, since there is only one available source host — a limitation of the environment rather than a finding about the activity itself.

---

## Verdict

**Classification: Consistent with Benign Activity — Not Conclusively Confirmed**

The observed pattern — a small number of failed attempts at human-typing cadence, a single consistent source, and an eventual successful login using the correct password — does not present any indicator that positively confirms malicious activity. However, this verdict is bounded by explicit limitations rather than high confidence:

- No historical baseline exists for this account to confirm the login falls within its normal pattern, as this is a freshly built lab environment.
- The lab's single-source topology removes source IP and geolocation as discriminating factors, both of which are primary real-world indicators for this exact scenario.
- No secondary telemetry (endpoint device history, VPN logs) was available to independently corroborate the account owner's identity.

In a production environment, this case would likely be closed as a low-priority false positive only after baseline and contextual checks confirm consistency with the account's normal behavior — not solely on the basis of the authentication event sequence alone.

---

## Recommended Runbook

**1. Immediate Triage**
- Compare the login's time, frequency of failures, and source against the account's historical baseline, if available. This is the fastest, lowest-cost check and should be performed first.
- If a baseline is unavailable or inconclusive, gather contextual enrichment: geolocation of the source IP (flags impossible-travel or unexpected-country logins), VPN logs (confirms expected connection path), and source device history (e.g., `WorkstationName` in the logon event, or endpoint management records) — a login from an unrecognized device is a red flag independent of source IP.

**2. Containment / Response**
- If the login is consistent with the account's established baseline and context, close as benign (false positive).
- If the login deviates from baseline, or no baseline exists to make a determination, treat as inconclusive and scale the response to risk: disable the account as a precautionary containment step if it has access to sensitive systems/data or if other suspicious signals are present (unusual time, unrecognized device, geolocation mismatch); treat as lower risk and proceed without immediate containment if the account is low-privilege and no other suspicious signals exist.
- Once contained (if applicable), contact the account owner directly to confirm whether the login activity was theirs — this is the only way to resolve identity ambiguity that technical telemetry alone cannot answer.

**3. Preventive / Policy**
- If confirmed or suspected malicious, notify a Tier 2 analyst, apply additional monitoring on the account's subsequent activity and any systems/data it accessed, and consider a forced password reset regardless of the final determination.
- Ensure Remote Desktop Users group membership is reviewed periodically, since authentication success does not guarantee — and should not be conflated with — authorization to establish a session.

---

## Notes / Limitations

- This case relies on manually reconstructed timing (30-second failure intervals, 4-minute gap before success) rather than organically occurring data, and should be read as a designed demonstration of an ambiguous pattern rather than a real detection.
- No historical baseline existed for the `test` account, which is the single largest limitation of this case; this gap is unavoidable in a built lab environment and is noted here rather than fabricated.
- The `test` account was granted Remote Desktop Users permissions specifically to allow this simulation to complete; this is treated narratively as a standard account with legitimate RDP access, and should not be read as demonstrating that domain-joined accounts have RDP access by default.
- This case demonstrates the authentication-vs-authorization distinction (LogonType 3 vs. 10) directly, extending the LogonType behavior first observed in Case 001.
