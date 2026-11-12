# Alert triage: Suricata "STREAM pkt seen on wrong thread"

603 medium alerts landed on the Elastic dashboard in a single day, all from Suricata. First thought was that the lab got popped again. It didn't. This is the triage from start to finish, including the part I nearly got wrong.

Same framework as Chapter 1: Categorize, Verify, Respond, Document.

**Verdict: false positive. Sensor noise, not an attack.**

## The alert

- 603 alerts, all Medium severity, risk score 47, all from Suricata.
- One catch-all rule behind them: "Suricata IDS alert (any signature)", which I set up to fire on every Suricata signature for triage practice.
- The last lab died to a botnet, so a spike like this gets the heart going. That is exactly why you work the process instead of guessing.

## Categorize

Filtered to outbound traffic first. Inbound scanner noise against an internet-facing box is constant and tells me nothing useful. What matters is my server reaching out, because that is what a compromise looks like. So I set the lab host as the source and looked at where it was talking to.

Then grouped the outbound alerts by destination. Most of it landed on a small set of addresses: the cloud metadata endpoint (169.254.169.254, link-local, roughly 47% of the volume) and a few Google, AWS, and Cloudflare addresses.

## Verify

Three checks, same as always.

**Who owns the IPs.** WHOIS on the top destinations came back as Google and other big cloud providers. Shared CDN and cloud space, not some obscure bulletproof host.

**Reputation.** VirusTotal was 0 of 92 vendors flagging the destination. One community commenter had tagged it as a malicious scanner IP, and AbuseIPDB had old reports on it with the confidence score decayed down to 0%. That is normal for recycled cloud IPs that a lot of people share. I went with the vendor consensus over one anonymous comment. A destination getting reported on shared cloud space does not mean my box did anything wrong.

**Behavior.** The alerts ticked on a clean rhythm, about every two and a half minutes, which looks like beaconing. But the same timestamps hit Google and AWS at the same instant, and a single command-and-control channel does not fan out to unrelated providers simultaneously. That broke the C2 theory.

Then the check that actually settled it. I pulled the raw JSON for one alert instead of trusting the dashboard summary, and the real signature was:

```
SURICATA STREAM pkt seen on wrong thread
```

That is not a threat detection. It is a Suricata engine health event. The sensor is reporting that a packet got handled by a different worker thread than the one that owns its flow, which is a sensor threading artifact and has nothing to do with an attacker. The rule description confirmed the setup: it fires on every Suricata signature, so an informational event like this one gets promoted straight to a Medium alert, 603 times over.

## Respond

Nothing to respond to on the security side, because there was no attack. The real action was on the tuning side.

- Closed the 603 alerts as a false positive.
- Flagged the rule for tuning. A catch-all "any signature" rule that turns informational engine events into Medium alerts is a noise machine. In a live SOC, 603 of these in a day would bury a real detection. The fix is a suppression or exception so those stream events stop being promoted, which is a detection-engineering job rather than a Tier 1 one.

## Document

**Alert:** Suricata IDS alert (any signature), signature "SURICATA STREAM pkt seen on wrong thread"

**Volume and severity:** 603, Medium

**Verdict:** False positive

**Reasoning:** Informational Suricata stream-engine event, not a detection. Destinations were clean cloud and CDN infrastructure on vendor consensus. Timing and multi-provider fan-out ruled out a single C2 channel. No sign of attacker activity.

**Action:** Closed as false positive. Recommended rule tuning to suppress the stream events.

**Escalation:** None. No malicious activity, tuning recommendation only.

## The honest part

This is the bit I want kept up front, because it is the real lesson.

Closing these alerts as noise is a statement about the alerts. It is not a clean bill of health for the host. I proved the detection was benign. I did not, on this pass, go to the host and positively identify which process was making the periodic outbound calls. That is a deeper investigation, and you only start it if triage turns up something that actually looks like an attack. Nothing here did.

If I were taking it further, the next step would be tying those outbound flows back to a local process to name exactly what is phoning out, most likely a monitoring or telemetry agent doing its normal job.

The one line I would underline for anyone starting out: an alert is not an attack, and read the raw event before you believe the summary.
