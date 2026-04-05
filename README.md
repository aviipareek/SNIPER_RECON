# SNIPER_RECON
The Elite Stealth Recon Playbook: A zero-touch, low-noise reconnaissance methodology for bug bounty hunters and red teamers. Maximize signal, minimize noise, and stay completely under the radar. 🥷💻

## 🚀 What's Inside?
This playbook covers a step-by-step pipeline:
* **Phase 1: Zero-Touch Seed Discovery** (Finding ASNs and CIDRs passively).
* **Phase 2: Stealth Subdomain Enumeration** (100% Passive API Pulls).
* **Phase 3: Ninja DNS Resolution** (Hiding behind public DNS resolvers).
* **Phase 4: Stealth Web Probing & WAF Evasion** (Origin IP hunting and Host Header forgery).
* **Phase 5: Passive Archive Extraction** (Scraping Wayback and AlienVault).
* **Phase 6: Local JavaScript Analysis** (Polite downloading and local secret hunting).
* **Phase 7: Targeted Sniper Fuzzing** (Low-rate, highly targeted endpoint discovery).

## 🛠️ Tools Utilized
`amass`, `subfinder`, `dnsx`, `httpx`, `waybackurls`, `gau`, `ffuf`, `nuclei`, `trufflehog`, `jq`, `curl`

---
*"Silence is the hacker's loudest weapon. Stay stealthy."*

# 🥷 THE ELITE STEALTH RECON PLAYBOOK (ARVIND'S SNIPER EDITION) 🥷

Welcome to the ultimate low-noise, high-impact reconnaissance methodology for Bug Bounty Hunters and Red Teamers. Too many hunters rely on noisy, automated scripts that trigger WAFs, burn IPs, and alert Blue Teams within minutes. This playbook is a calculated, 7-phase approach to mapping an organization's entire external infrastructure while remaining completely invisible for as long as possible.

## 🎯 The 3 Golden Rules of Sniper Recon
> **Rule #1:** Touch the target server as little as possible.
> **Rule #2:** Blend in with normal traffic.
> **Rule #3:** Parse data locally, not on the target's server.

## 🛠️ Prerequisites (Tools Required)
Ensure you have the following tools installed and configured in your system:
`amass`, `subfinder`, `dnsx`, `httpx`, `waybackurls`, `gau`, `ffuf`, `nuclei`, `trufflehog`, `gf` (with aws & sec patterns), `jq`, `curl`, `wget`.

---

## 📡 PHASE 1: ZERO-TOUCH SEED DISCOVERY (ASN & Reverse WHOIS)
**Goal:** Find the company's IP blocks without sending a single packet to them.

1. **Find ASN via Hurricane Electric (BGP):**
```bash
curl -s "[https://bgp.he.net/search?search%5Bsearch%5D=CompanyName&commit=Search](https://bgp.he.net/search?search%5Bsearch%5D=CompanyName&commit=Search)" | grep -Eo "AS[0-9]+" | sort -u > asns.txt
```

2. **Get CIDR (IP Ranges) from ASN passively:**
```bash
whois -h whois.radb.net -- '-i origin AS<ASN_NUMBER>' | grep -Eo "([0-9.]+){4}/[0-9]+" | sort -u > cidr.txt
```

3. **Find Root Domains via Reverse WHOIS (Passive):**
```bash
amass intel -whois -d example.com > root_domains.txt
```

---

## 🕵️‍♂️ PHASE 2: STEALTH SUBDOMAIN ENUMERATION (100% Passive API Pulls)
**Goal:** Collect subdomains from 3rd-party databases, NOT the target.

1. **Subfinder (Passive mode only, low threads):**
```bash
subfinder -d example.com -all -t 10 -silent > subfinder.txt
```

2. **Amass (Strictly Passive):**
```bash
amass enum -passive -norecursive -d example.com -o amass.txt
```

3. **Certificate Transparency (crt.sh):**
```bash
curl -s "[https://crt.sh/?q=%25.example.com&output=json](https://crt.sh/?q=%25.example.com&output=json)" | jq -r '.[].name_value' | sed 's/\*\.//g' | sort -u > crtsh_subs.txt
```

4. **GitHub Passive Dorking (Find hidden internal/dev URLs):**
```bash
github-endpoints -d example.com -t <YOUR_GITHUB_TOKEN> -o github_subs.txt
```

5. **Clean & Merge (Local execution):**
```bash
cat subfinder.txt amass.txt crtsh_subs.txt github_subs.txt | sort -u > all_subs.txt
```

---

## 🥷 PHASE 3: NINJA DNS RESOLUTION (Hiding behind Public Resolvers)
**Goal:** Check which subs are alive, but route queries through public DNS (`8.8.8.8`) so the target doesn't see our IP.

1. **Download fresh Public Resolvers:**
```bash
wget [https://raw.githubusercontent.com/trickest/resolvers/main/resolvers.txt](https://raw.githubusercontent.com/trickest/resolvers/main/resolvers.txt) -O resolvers.txt
```

2. **Resolve via Public DNS (Low rate to avoid dropping packets):**
```bash
dnsx -l all_subs.txt -r resolvers.txt -t 50 -rl 100 -a -resp -o resolved_subs_ips.txt
cat resolved_subs_ips.txt | awk '{print $1}' > alive_subs_only.txt
```

---

## 🌐 PHASE 4: STEALTH WEB PROBING & WAF EVASION
**Goal:** Find HTTP/HTTPS servers blending in as a normal Google Chrome user.

1. **HTTPX Probing (Random User-Agents, Delay, Low Threads):**
```bash
cat alive_subs_only.txt | httpx -sc -title -tech-detect -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0" -rate-limit 50 -t 20 -o alive_web.txt
```

2. **Origin IP Hunt via Shodan/Censys (Zero-Touch WAF Bypass):**
*(Never port-scan Cloudflare IPs. Find the origin passively.)*
```text
Shodan Dork: ssl.cert.subject.CN:"*.example.com" 200
Censys Dork: services.tls.certificates.leaf_data.subject.CN: "*.example.com"
```

3. **Host Header Forgery (Testing the Origin IP):**
```bash
curl -ik -H "Host: hidden-api.example.com" https://<ORIGIN_IP>/
```

---

## 🕰️ PHASE 5: PASSIVE ARCHIVE EXTRACTION (Scraping the Past)
**Goal:** Find old, forgotten endpoints without touching the live server.

1. **Fetch Wayback & AlienVault URLs (Zero-Touch):**
```bash
echo "example.com" | waybackurls > wayback_urls.txt
echo "example.com" | gau --subs > gau_urls.txt
```

2. **Filter out junk locally (Images, CSS, Fonts):**
```bash
cat wayback_urls.txt gau_urls.txt | sort -u | grep -ivE "\.(jpg|jpeg|gif|css|tif|tiff|png|ttf|woff|woff2|ico)$" > clean_urls.txt
```

---

## 📜 PHASE 6: LOCAL JAVASCRIPT ANALYSIS (The Goldmine)
**Goal:** Download JS files once and analyze them locally to avoid rate-limits.

1. **Extract JS URLs:**
```bash
cat clean_urls.txt | grep "\.js$" | sort -u > js_urls.txt
```

2. **Download files locally (Slow & Polite download):**
```bash
mkdir js_files && cd js_files
wget -i ../js_urls.txt --wait=1 --random-wait --user-agent="Mozilla/5.0"
```

3. **Local Secret Scanning (Grep for keys without network traffic):**
```bash
gf aws-keys *.js > aws_leaks.txt
gf sec *.js > secrets.txt
trufflehog filesystem ./js_files
```

---

## 🎯 PHASE 7: TARGETED SNIPER FUZZING (No "Spray and Pray")
**Goal:** Fuzz only highly suspicious endpoints with high delays.

1. **Polite API Fuzzing (Delay `-p 0.5` seconds between requests):**
*Tip: For stealth fuzzing, avoid massive wordlists. Use targeted ones like `raft-small-directories.txt`.*
```bash
ffuf -w /Path/to/wordlists/ -u [https://api.example.com/v1/FUZZ](https://api.example.com/v1/FUZZ) -H "User-Agent: Mozilla/5.0 Chrome/120" -p 0.5 -mc 200,301
```

2. **Targeted Vulnerability Scanning (Nuclei in Sniper Mode):**
*(Run only critical/exposure templates, limit to 10 req/sec.)*
```bash
nuclei -l alive_web.txt -tags exposures,cve,misconfig -rl 10 -c 5 -o sniper_nuclei.txt
```

---

## ⚠️ Legal Disclaimer
This playbook is strictly for educational purposes and authorized bug bounty programs only. Do not use these techniques on targets you do not have explicit, written permission to test. You are solely responsible for your actions.

### 💻 SILENCE IS THE HACKER'S LOUDEST WEAPON. STAY STEALTHY. 🥷
