# TryHackMe Progress

## Pre Security Path
- [x] Room 1: Introduction to Cyber Security
  
Overview room (no hands-on tasks).
  Covered the difference between offensive security (attacking systems to
  find weaknesses, e.g. penetration testing) and defensive security
  (protecting systems and responding to attacks, e.g. SOC analyst, incident
  response). Also introduced different career paths within cybersecurity.
- [x] Room 2: Offensive Security Intro

Key concepts covered:
- Introduction to offensive security and how it differs from defensive security
- Overview of the penetration testing process: reconnaissance → discovery → exploitation
- Directory enumeration using Gobuster to discover hidden pages/endpoints on a
  web application
- Practical exploitation exercise on a simulated vulnerable banking site

### Lab: Fake Banking Site (fakebank.thm)

### Step 1: Directory Enumeration
Used Gobuster to brute-force hidden directories on the target site: 

gobuster dir -u http://fakebank.thm -w wordlist.txt

This command scans the target URL and checks it against a wordlist of common
directory/file names, revealing pages not linked anywhere on the visible site
(a common way real hidden admin panels or unprotected pages are discovered).

### Step 2: Exploitation
Discovered a hidden endpoint through enumeration and used it to access
functionality that should have required proper authentication — successfully
transferred funds into my own account within the simulated banking application.

## What I Learned
This lab demonstrated a real-world vulnerability class: broken access control /
insecure direct object exposure, where an application exposes sensitive
functionality (like fund transfers) through hidden or unlinked pages instead of
through properly authenticated, authorized routes. Gobuster proved how
attackers routinely find these "hidden" pages that developers assume are safe
simply because they aren't linked in the UI.
