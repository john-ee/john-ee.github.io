---
title: "Understanding MS-MDE2: The Protocol Behind Intune Enrollment"
date: 2025-07-13 10:00:00 +0000
categories: [MDM, Windows]
tags: [ms-mde2, oma-dm]
draft: true
---

# What is MS-MDE2: The Protocol Behind Intune Enrollment

## 1. Intro
As an Endpoint Engineer, I work in IT takeover projects during company acquisitions.
One of the migration strategies is to move the device to the client's management suite as is with a re-ACL to the local session so it belongs to the new domain account. The user keeps their existing session, they simply re-authenticate with their new credentials. 
This strategy works in on-premise settings - device management done with an Active Directory and Group Policies. But it fails if the device is already Intune-enrolled. Why ?

The fix was to perform the following actions before migrating the device :
- Identifying the scheduled tasks in the path **Task Scheduler Library > Microsoft > Windows > EnterpriseMgmt** and noting down the GUIDs
    - The tasks run actions such as a PushLaunch and PushRenewal which interact with Intune
- Deleting registry keys identified with the same GUIDs as above in the path **HKLM\SOFTWARE\Microsoft\Enrollments**
    - The registry keys identify the device to the proper tenant.
- Deleting the client certificate issued by **Microsoft Intune MDM Device CA**
    - The identity of the device.

This [archived article](https://web.archive.org/web/20231001060136/https://raymonddewit.com/manually-re-enrollment-of-a-windows-10-11-pc-in-intune/) gave me the tip. If you work with Intune, I strongly recommend to check out those paths on a test device. 

How are these artifacts set during enrollment ?

## 2. What is MS-MDE2 ?
According to Microsoft Learn, *The Mobile Device Enrollment (MDE) protocol enables a device to be enrolled with a Device Management Service (DMS) through an Enrollment Service (ES), including the discovery of the Management Enrollment Service (MES) and enrollment with the ES. After a device is enrolled, the device can be managed with the DMS using MDM.* [MS-MDE2 Overview](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-mde2/98547779-b770-4730-9261-8ecaa1604c10)
In our case, the DMS is Intune. The Enrollment Service itself uses the protocols MS-XCEP and MS-WSTEP
**MS-XCEP** is a minimal messaging protocol that explains the rules for requesting the certificate.
**MS-WSTEP** is the protocol for the certificate issuance.
Finally, **MS-MDM** is the protocol that takes over once enrollment is complete. It governs the ongoing commands (policy pushes, queries, config) exchanged between Intune and the device.

## 3. Enrollment Flow
- [Discovery phase]
- [Get Security Token]
- [Get Policies]
- [Request Security Token]
- [Provisioning handoff]
- [diagram placeholder]

## 4. How You Can Leverage It for Your Own Self-Hosted / Self-Built MDM
- [what components you'd need to stand up: DS, ES, MES endpoints]
- [what Intune abstracts away that you'd have to build yourself]
- [certificate/token handling considerations]
- [gotchas / where the spec is ambiguous or underdocumented]
- [closing: link to spec, maybe a call-to-action]