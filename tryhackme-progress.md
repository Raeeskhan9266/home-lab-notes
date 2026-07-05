# TryHackMe Progress

## Pre Security Path
## **Module 1: Introduction to Cyber Security**
  
Overview (no hands-on tasks).
  Covered the difference between offensive security (attacking systems to
  find weaknesses, e.g. penetration testing) and defensive security
  (protecting systems and responding to attacks, e.g. SOC analyst, incident
  response). Also introduced different career paths within cybersecurity.
  
- [x] **Room 1: Offensive Security Intro**

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

- [x]  **Room 2: Defensive Security Intro**

Key concepts covered:
- Introduction to defensive security and how it complements offensive security
  (protecting and monitoring systems vs. actively attacking them to find flaws)
- Security Operations Center (SOC): role of a SOC team in continuously
  monitoring and responding to security incidents
- Threat Intelligence: gathering and using information about attackers,
  their methods, and indicators of compromise to anticipate and prevent attacks
- Digital Forensics and Incident Response (DFIR): investigating security
  incidents after they occur, preserving evidence, and understanding how a
  breach happened
- Malware Analysis: examining malicious software to understand its behavior,
  purpose, and impact

## Lab: SIEM Tool Demo

Used a SIEM (Security Information and Event Management) tool to observe and
respond to a simulated security incident:
- Identified an unauthorized login/access attempt through an alert generated
  by the SIEM
- Traced the alert to the source IP address responsible for the attempt
- Blocked the malicious IP address using a demo firewall interface, stopping
  further unauthorized access

## What I Learned
This room gave me the "other side" of the picture compared to my offensive
work so far. Where my home lab exercises (Metasploitable2, Gobuster) focused
on finding and exploiting weaknesses, this room showed how defenders detect
and respond to those same kinds of attacks in real time — a SIEM tool
essentially aggregates logs and alerts across a network so a SOC analyst can
spot suspicious activity (like repeated failed logins or unusual access
patterns) and act on it quickly, in this case by blocking the offending IP
at the firewall.

Understanding both sides — how attacks are carried out and how they're
detected/blocked — gives a more complete picture of real-world security work,
and reinforces why fast detection and response matters as much as prevention.

- [x]  **Room 3: Careers in Cyber**

## Overview
This room covered an overview of different career paths within cybersecurity,
along with the core responsibilities of each role:

- **Security Analyst** — Monitors systems and networks for suspicious activity,
  analyzes alerts, and investigates potential security incidents
- **Security Engineer** — Designs, builds, and maintains security systems and
  infrastructure (firewalls, security tools, secure architecture)
- **Incident Responder** — Responds to active security breaches, contains
  damage, and manages the process of recovering from an attack
- **Digital Forensics Examiner** — Investigates systems after an incident to
  collect and preserve evidence, and reconstructs how an attack happened
- **Malware Analyst** — Reverse-engineers and studies malicious software to
  understand its behavior, origin, and impact
- **Penetration Tester** — Simulates real-world attacks against systems and
  applications (with authorization) to find vulnerabilities before malicious
  actors do
- **Red Teamer** — Simulates full-scale, realistic adversary attacks against
  an organization to test detection and response capabilities across people,
  process, and technology — broader in scope than standard penetration testing

## What I Learned
This room helped me understand how the different roles in cybersecurity
connect to each other — offensive roles (Penetration Tester, Red Teamer) find
and exploit weaknesses, while defensive roles (Security Analyst, Incident
Responder, Digital Forensics Examiner) detect, respond to, and investigate
those same attacks. Malware Analyst and Security Engineer sit slightly
adjacent to both, focusing on understanding threats and building resilient
systems respectively.

