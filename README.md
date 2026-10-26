# PortSwigger Web Security Academy — Lab Writeups

Hands-on writeups from the PortSwigger Web Security Academy, focused on authentication vulnerabilities as part of my transition from QA/software testing into Application Security.

**Tools used:** Burp Suite Community Edition (Intruder, Comparer, Repeater)

---

## Authentication

### 1. Username enumeration via different responses
**Status:** ✅ Solved

**Vulnerability:** The login endpoint returned different HTTP response lengths depending on whether the submitted username existed, allowing an attacker to enumerate valid usernames without any credentials.

**Approach:**
1. Captured a login request in Burp Proxy and sent it to Intruder.
2. Marked the `username` parameter as the payload position (Sniper attack).
3. Loaded the candidate usernames wordlist provided by the lab.
4. Ran the attack and sorted results by response `Length` — one row stood out from the rest, indicating a valid username.
5. Repeated the attack with the username fixed and the `password` parameter marked as the payload position, using the candidate passwords wordlist.
6. Identified the correct password by status code: all incorrect attempts returned `200`, while the correct password returned `302` (redirect to the account page on successful login).

**Key takeaway:** Login forms must return identical responses (same wording, length, and status code) regardless of whether the username is valid, to prevent enumeration.

---

### 2. Username enumeration via subtly different responses
**Status:** ✅ Solved

**Vulnerability:** Same underlying flaw as Lab 1, but the application added randomized padding to every response, making simple `Content-Length` comparison unreliable.

**Approach:**
1. Ran the same username-enumeration Intruder attack as Lab 1.
2. Noticed response lengths clustered into noisy bands rather than showing one clean outlier — confirmed via Burp **Comparer** that two different-length responses only differed in a randomly generated analytics ID injected into the page, not in actual content.
3. Since response length was unreliable, used Burp Intruder's **Grep - Match** feature to search every response for the literal phrase `"Invalid username or password"`.
4. All but one response matched that phrase — the single non-matching response indicated a subtle, invisible wording difference, revealing the valid username.
5. Brute-forced the password the same way as Lab 1 (status code `302` = success).

**Key takeaway:** Randomized padding alone isn't enough to prevent enumeration if the underlying response content still differs — attackers can isolate the real signal from noise using content-matching instead of raw length comparison.

---

### 3. Username enumeration via response timing
**Status:** 🔄 In progress

**Vulnerability:** The application takes measurably longer to respond when a submitted username is valid, because the backend only performs password hashing if the username exists in the system.

**Approach so far:**
1. Calibrated the timing signal manually: submitted a known-valid username (`wiener`) with a very long password (1000+ characters) and compared its response time against an invalid username with the same long password — confirmed a large timing gap (~2300ms vs ~140ms), proving the technique works on this lab instance.
2. Ran the full candidate username list through Intruder (Sniper attack) using the long password, sorting results by response time (`Response received` column in Burp).
3. Identified an outlier username with a response time roughly 3x higher than the rest of the field.
4. Currently brute-forcing the password for the confirmed username (in progress — working around a temporary account lockout triggered during testing).

**Key takeaway:** Even when response content and length are identical, backend processing time differences (e.g., expensive hashing operations) can leak information. Using a deliberately long password amplifies this timing gap, making it detectable over normal network jitter.

---

## Notes on methodology

- All labs solved using Burp Suite **Community Edition** (no Pro features used).
- Where a single technique failed (e.g. raw response length), pivoted to a different signal (content matching, response timing) rather than treating it as a dead end — reflects the general approach of layering multiple detection techniques when the obvious one is deliberately obscured.
