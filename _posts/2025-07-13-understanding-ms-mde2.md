---
title: "What is MS-MDE2: The Protocol Behind Intune Enrollment"
date: 2025-07-13 10:00:00 +0000
categories: [MDM, Windows, Troubleshooting]
tags: [ms-mde2, oma-dm]
draft: true
---

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
According to Microsoft Learn,
> The Mobile Device Enrollment (MDE) protocol enables a device to be enrolled with a Device Management Service (DMS) through an Enrollment Service (ES), including the discovery of the Management Enrollment Service (MES) and enrollment with the ES. After a device is enrolled, the device can be managed with the DMS using MDM. [MS-MDE2 Overview](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-mde2/98547779-b770-4730-9261-8ecaa1604c10)

In our case, the DMS is Intune. The Enrollment Service itself uses the protocols MS-XCEP and MS-WSTEP.
**[MS-XCEP](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-xcep/47ea22c9-e5d7-4c40-a9c6-3407c9d5f31c)** is a minimal messaging protocol that explains the rules for requesting the certificate.
**[MS-WSTEP](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-wstep/ac55b8cc-9ade-4982-b135-991d574ade74)** is the protocol for the certificate issuance.
Finally, **[MS-MDM](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-mdm/d8ef69b5-0d09-4ecc-8154-977586b764c6)** is the protocol that takes over once enrollment is complete. It governs the ongoing commands (policy pushes, queries, config) exchanged between the MDS and the device.

## 3. Enrollment Flow
This figure from Microsoft Learn summarizes the enrollment flow.

![Typical_sequence_for_enrolling_a_message_using_MDE2](/assets/img/2025-07-13-understanding-ms-mde2/Typical_sequence_for_enrolling_a_message_using_MDE2.png)

_Typical sequence for enrolling a message using MDE2 (from Microsoft Learn)_

1. The user's email name resolves the proper Discovery Service. If your company Contoso owns the domain contoso.org and tied it to your tenant, that's how your device will be joined to your Intune service and not another's. If you inspected the registry keys mentioned earlier, you will notice the key **DiscoveryServiceFullURL**.
2. The client queries the Security Token Service. Depending on the service, your client might need to authenticate with its UserPrincipalName and password, which the user provides if they connect with their professional access on a personal device or they log in during the Autopilot OOBE.
3. Now we get into the protocols we identified earlier. The client requests the server about the certificates (MS-XCEP) then plays by the rules. The client generates a Certificate Signing Request, sends it to the server and behind the scenes, a Certificate Authority signs. That's where the client certificate comes from.
4. Now your device is enrolled and uses the **[MS-MDM](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-mdm/33769a92-ac31-47ef-ae7b-dc8501f7104f)** protocol for its management. This is the one that configures your device if you deploy Configuration Profiles.

We figured out where the certificate comes from. When do the registry keys and the scheduled tasks come in ?
The articles I consulted do not mention these artifacts. I've confirmed the timing of the creation of scheduled tasks match the device enrollment's timestamp. The reason I do not find anything specific is because the protocol's documentation is about the channel itself, between the server and the client. The queries, the answers, the format the messages need to have.

## 4. How you can leverage it for your own Self-Hosted / Self-Built MDM
To build your own Windows Device Management suite, you will need to implement :
- The MS-MDE2 protocol in full (ES, MS-XCEP, MS-WSTEP) for the enrollment
- The MS-MDM protocol to manage the device. CSPs and OMA-URI are worth a discussion on their own.
- A valid certificate or a self signed certificate deployed to the client before enrollment
- A Certificate Authority

LLM-assistance makes writing code much easier, so I have a drafted code here : [win-dm-lite](https://github.com/john-ee/win-dm-lite)
The goal is to inspect what happens on the client, to simulate enrollment and management and finally have a proof-of-concept with an actual Windows client.

## Bibliography

#### Primary sources (Microsoft Learn – Open Specifications):
- Microsoft. [MS-MDE2]: Mobile Device Enrollment Protocol Version 2 – Overview. Microsoft Learn. https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-mde2/98547779-b770-4730-9261-8ecaa1604c10
- Microsoft. [MS-XCEP]: X.509 Certificate Enrollment Policy Protocol – Overview. Microsoft Learn. https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-xcep/47ea22c9-e5d7-4c40-a9c6-3407c9d5f31c
- Microsoft. [MS-WSTEP]: WS-Trust X.509v3 Token Enrollment Extensions – Overview. Microsoft Learn. https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-wstep/ac55b8cc-9ade-4982-b135-991d574ade74
- Microsoft. [MS-MDM]: Mobile Device Management Protocol – Overview. Microsoft Learn. https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-mdm/d8ef69b5-0d09-4ecc-8154-977586b764c6
- Microsoft. [MS-MDM]: Mobile Device Management Protocol – Introduction. Microsoft Learn. https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-mdm/33769a92-ac31-47ef-ae7b-dc8501f7104f

#### Practitioner source:
- de Wit, Raymond. Manually Re-Enrollment of a Windows 10/11 PC in Intune. Archived via Wayback Machine, October 1, 2023. https://web.archive.org/web/20231001060136/https://raymonddewit.com/manually-re-enrollment-of-a-windows-10-11-pc-in-intune/

#### Related project:
- win-dm-lite — proof-of-concept repository. GitHub. https://github.com/john-ee/win-dm-lite
