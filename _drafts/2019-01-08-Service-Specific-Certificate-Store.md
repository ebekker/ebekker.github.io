---
published: false
layout: post
title: Service-Specific Certificate Store
---

Recently I ran into situation where I needed to access the Certificate Store *scoped* to a specific Windows Service.  If you're not familiar with this, the Certificate Store on Windows can be scoped to different levels.  There's the global store that's scoped to `LocalMachine`, then there are user-specific stores which default to the `CurrentUser`.

Additionally, since at least Windows 2008, Certificate Store access can be scoped to a specific Windows Service.  This can be useful if you want to compartmentalize and isolate certain certificates to certain services.  For some services, this may even enable certain unique features, which is the use case that brought me here.

In this instance, I was looking to secure LDAP access to a Windows Domain over TLS (LDAPS).  To do so means providing a valid certificate with certain characteristics to the `NTDS` Windows Service (Active Directory Domain Services), as documented [here](https://support.microsoft.com/en-us/help/321051/how-to-enable-ldap-over-ssl-with-a-third-party-certification-authority).  The documentation further states:

> AD DS preferentially looks for certificates in this store over the Local Machine's store.

and

> AD DS detects when a new certificate is dropped into its certificate store and then triggers an SSL certificate update without having to restart AD DS or restart the domain controller.

Nice!  Updating the certificate used to secure the LDAPS interface without a server, or even a service restart.

