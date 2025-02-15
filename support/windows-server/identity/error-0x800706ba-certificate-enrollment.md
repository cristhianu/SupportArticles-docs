---
title: Error 0x800706ba during certificate enrollment
description: Introduces steps to resolve the error 0x800706ba, The RPC Server is unavailable, which occurs during certificate enrollment.
author: Deland-Han
ms.author: delhan
ms.topic: troubleshooting
ms.date: 05/17/2023
ms.reviewer: kaushika
ms.prod: windows-server
ms.technology: windows-server-active-directory
ms.custom: sap:active-directory-certificate-services, csstroubleshoot, ikb2lmc
---
# Error 0x800706ba "The RPC Server is unavailable" when you enroll a certificate

When you try to enroll a certificate on a Windows Server, it fails with the error 0x800706ba, "The RPC Server is unavailable." This article introduces steps to resolve this issue.

_Applies to:_ Supported versions of Windows Server  
_Original KB number:_ 4042719, 4516764, 5021150  

## Identify the issue

When you encounter this issue, you may see one or more of the following symptoms.

> [!NOTE]
> When the issue occurs, if we add the user account that's used to request the certificate to the local administrators group on the certificate authority (CA), the enrollment succeeds for a user-based template. However, enrollment against a machine-based template still returns the same error.

### Error messages

You receive error messages that resemble the following during certificate enrollment.

:::image type="content" source="media/error-0x800706ba-certificate-enrollment/progress-window-of-certificate-enrollment.png" alt-text="Screenshot that shows the progress window of certificate enrollment." border="true":::

:::image type="content" source="media/error-0x800706ba-certificate-enrollment/error-message-during-certificate-enrollment.png" alt-text="Screenshot that shows the error message during certificate enrollment." border="true":::

### Network capture

The network trace shows the successful Lightweight Directory Access Protocol (LDAP) queries to the configuration partition in Active Directory; the templates available are revealed in the trace.

Then, the requesting server tries to do a remote procedure call (RPC) to the CA and gets the response "ACCESS DENIED."

For example:

> 10167 \<time\> \<requesting server IP address\> \<CA IP address\> ISystemActivator 918 RemoteCreateInstance request  
> 10174 \<time\> \<CA IP address\> \<requesting server IP address\> DCERPC 86 Fault: call_id: 3, Fragment: Single, Ctx: 1, **status: nca_s_fault_access_denied**

Additionally, you can find the Microsoft Remote Procedure Call (MSRPC) bind attempt and error:

> 1093    \<time\>    92.5590216     (0)    SourceIP    52237 (0xCC0D)    DestIP    135 (0x87)    MSRPC    MSRPC:c/o Bind: IRemoteSCMActivator(DCOM) UUID{000001A0-0000-0000-C000-000000000046}  Call=0x3  Assoc Grp=0x8A9E  Xmit=0x16D0  Recv=0x16D0  
> 1097    \<time\>    92.5940283     (652)    SourceIP    135 (0x87)    DestIP    52237 (0xCC0D)    MSRPC    MSRPC:c/o Bind Nack:  Call=0x3  **Reject Reason: invalid_checksum**

### Event log

If auditing is enabled, a Distributed Component Object Model (DCOM) error can be observed on the CA server detailing an **ANONYMOUS LOGON** attempt:

```output
Log Name: System  
Source: Microsoft-Windows-DistributedCOM  
Date: \<date\>  
Event ID: 10027  
Task Category: None  
Level: Warning  
Keywords: Classic  
User: ANONYMOUS LOGON  
Computer: \<CA FQDN\>

Description:  
The machine wide limit settings do not grant Remote Activation permission for COM Server applications to the user NT AUTHORITY\ANONYMOUS LOGON SID (S-1-5-7) from address \<IP address\> running in the application container Unavailable SID (Unavailable). This security permission can be modified using the Component Services administrative tool.
```

### Other symptoms and logs

- The call should be made with **dce_c_authn_level_pkt_integrity** RPC Integrity level that enforces Kerberos or New Technology LAN Manager (NTLM) as an authentication mechanism. This behavior is enforced by default starting 6B.22 [KB5004442—Manage changes for Windows DCOM Server Security Feature Bypass (CVE-2021-26414)](https://support.microsoft.com/topic/kb5004442-manage-changes-for-windows-dcom-server-security-feature-bypass-cve-2021-26414-f1400b52-c141-43d2-941e-37ed901c769c).
- When the client sends a KRB_AP_REQ request, it's rejected by the server side.
- The server tries to procure an access token for the user who presented the Kerberos Ticket Granting Service (TGS) and fails with error 0xc000015b, "STATUS_LOGON_TYPE_NOT_GRANTED."

## Cause

This issue occurs because the group policy **Access this computer from the network** is set, and the user account used to enroll the certificate isn't added.

The group policy locates at: _Computer Configuration\\Windows Settings\\Security Settings\\Local Policies\\User Rights Assignment_.

By default, the policy is populated by the groups: Administrators, Backup Operators, Everyone, and Users.

Because the user account that's used for certificate enrollment fails authentication by using Kerberos, the authentication mechanism is downgraded to "anonymous logon." The logon fails on the DCOM level.

## Resolution

To resolve this issue, add the appropriate user groups to the group policy. For example:

:::image type="content" source="media/error-0x800706ba-certificate-enrollment/properties-of-access-this-computer-from-the-network.png" alt-text="Screenshot that shows the properties window of the 'Access this computer from the network' group policy.":::
