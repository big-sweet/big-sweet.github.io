---
layout: post
title: Mandatory LDAP Signing and LDAP Channel Binding Requirement
date: 2019-11-06
author: bigsweet
---
Microsoft has announced that in mid-March 2020 they will be forcing LDAP channel binding and LDAP signing. 

This is due to an exploit that was discovered which would allow for an attacker to elevate their privileges if an unsecured LDAP request was able to be intercepted by way of man-in-the-middle attack.

More information about the advisory can be found [here](https://support.microsoft.com/en-ca/help/4520412/2020-ldap-channel-binding-and-ldap-signing-requirement-for-windows)

There's potential for this update to cause problems in your environment if applications are not configured to make hardened requests to your domain controllers. 
## Discovering the Volume of Requests
By default a domain controller will log Event ID: 2887.

This log entry will list the number of simple binds performed without SSL/TLS and the number of Negotiate/Kerberos/NTLM/Digest binds performed without signing in the past 24 hours.

The event log will display as such:
```
Event ID: 2887
During the previous 24 hour period, some clients attempted to perform LDAP binds that were either:
(1) A SASL (Negotiate, Kerberos, NTLM, or Digest) LDAP bind that did not request signing (integrity validation), or
(2) A LDAP simple bind that was performed on a cleartext (non-SSL/TLS-encrypted) connection

This directory server is not currently configured to reject such binds. The security of this directory server can be significantly enhanced by configuring the server to reject such binds. For more details and information on how to make this configuration change to the server, please see http://go.microsoft.com/fwlink/?LinkID=87923.

Summary information on the number of these binds received within the past 24 hours is below.

You can enable additional logging to log an event each time a client makes such a bind, including information on which client made the bind. To do so, please raise the setting for the "LDAP Interface Events" event logging category to level 2 or higher.

Number of simple binds performed without SSL/TLS: 11
Number of Negotiate/Kerberos/NTLM/Digest binds performed without signing: 0
```
Note the two lines at the bottom which will tell you the number of requests, this is important for the next step.

## Identifying Affected Services and Applications
Of course you'll want to identify which hosts are needing remediation before patching your domain controllers. Otherwise you're going to end up with services and applications that will not authenticate properly to your domain.
### Increasing LDAP Interface Diagnostic Levels
**WARNING**: This setting has the capability to generate a tremendous amount of logs. If the number of requests being made from the last two lines of the Event ID 2887 log is high, consider only turning on this setting for a fixed amount of time (10 minutes should do).

```
# Enable Simple LDAP Bind Logging 
Reg Add HKLM\SYSTEM\CurrentControlSet\Services\NTDS\Diagnostics /v "16 LDAP Interface Events" /t REG_DWORD /d 2
```

```
# Disable Simple LDAP Bind Logging.
Reg Add HKLM\SYSTEM\CurrentControlSet\Services\NTDS\Diagnostics /v "16 LDAP Interface Events" /t REG_DWORD /d 0
```
Once this setting has been turned on and the Event ID 2889 logs are generating, it is time to capture and view the results.
### Viewing the Results
There are two options to view the generated logs, which you pick is entirely up to your preference.

Luckily, a lovely Microsoft PFE (Premier Field Engineer) by the name of Russell Tomkins has us covered.

#### Option 1: PowerShell
Russell's PowerShell script can be located on GitHub [here](https://github.com/russelltomkins/Active-Directory/blob/master/Query-InsecureLDAPBinds.ps1).

Once executed this script will output a CSV file containing information about the hosts sending insecure requests to your DC's.

#### Option 2: Event Viewer Custom Filter
Russell's XML can be found on GitHub [here](https://github.com/russelltomkins/Active-Directory/blob/master/LDAP%20Signing%20Events%20Custom%20View.xml)

1. Open Event Viewer.
2. Right-click "Custom Views" then select "Import Custom View".
3. Provide a path to the downloaded .xml file. Feel free to rename it/change the description and location as you see fit.
4. If you are doing this on a management server (and you should be) you will get an error about the service channel being missing. Once you bind to the appropriate Domain Controller, it will show the appropriate events.

## Remediating your Applications

So you've discovered a few applications or services that are communicating with your DC insecurely.

Usually this is as simple as going in to the application settings and enabling "Secure connection" or "Secure bind" where the application is configured to connect to your DC.

Some applications may require the use of certificates on your DC to allows TLS binds over port 636.

If your application does not allow for secure requests to be made to your DC, I would strongly advise replacing the application.