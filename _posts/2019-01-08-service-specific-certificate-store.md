---
published: true
layout: post
title: Service-Specific Certificate Store
---

Recently I ran into situation where I needed to access the Certificate Store *scoped* to a specific Windows Service.  If you're not familiar with this, the Certificate Store on Windows can be scoped to different levels.  There's the global store that's scoped to `LocalMachine`, then there are user-specific stores which default to the `CurrentUser`.

Additionally, since at least Windows 2008, Certificate Store access can be scoped to a specific Windows Service.  This can be useful if you want to compartmentalize and isolate certain certificates to certain services.  For some services, this may even enable certain unique features, which is the use case that brought me here.

In this instance, I was looking to secure LDAP access to a Windows Domain over TLS (LDAPS).  To do so means providing a valid certificate with certain characteristics to the `NTDS` Windows Service (Active Directory Domain Services), as documented [here](https://support.microsoft.com/en-us/help/321051/how-to-enable-ldap-over-ssl-with-a-third-party-certification-authority).  The documentation further states:

> AD DS preferentially looks for certificates in this store over the Local Machine's store.

and

> AD DS detects when a new certificate is dropped into its certificate store and then triggers an SSL certificate update without having to restart AD DS or restart the domain controller.

Nice!  Updating the certificate used to secure the LDAPS interface without a server, or even a service restart!  But therein lies the the rub -- all the documentation I've found to install a certificate into a service-specific certificate store always described the process using the graphical MMC tool, such as [this one](https://social.technet.microsoft.com/wiki/contents/articles/2980.ldap-over-ssl-ldaps-certificate.aspx#Exporting_the_LDAPS_Certificate_and_Importing_for_use_with_AD_DS) and [this one](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2008-R2-and-2008/dd941846(v=ws.10)#to-import-a-certificate-into-the-ad-ds-personal-store).

Just a quick recap:

* fire up MMC
* select **File > Add/Remove Snap-in**
* add the **Certificates** snap-in
  * select **Service account** to manage
  * select **Local computer**
  * select **Active Directory Domain Services** in the service account list

Pretty painless as far as GUI operations go, but in a perfect world we should be able to *automate all the things* and anything that requires manual interaction with the MMC is a road block.

A quick search for any way to manage service-specific certs in a scriptable fashion indicates that no such facility exists, such as [this](https://stackoverflow.com/questions/5149607/how-to-open-a-windows-service-certificate-store) and [this](https://social.technet.microsoft.com/Forums/en-US/07be0526-a8e7-4453-a1b0-910ff959758b/import-certificate-into-service-accounts-personal-store?forum=winserversecurity).

While Windows PowerShell does provide access to the Windows Certificate Store in the form of a [Certificate Provider](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.security/about/about_certificate_provider), it's restricted to only accessing `LocalMachine` and `CurrentUser` store locations.  This makes sense since PowerShell is making use of the facilities that are already available in the unerdlying .NET platform, and a quick look at the [StoreLocation](https://docs.microsoft.com/en-us/dotnet/api/system.security.cryptography.x509certificates.storelocation) enumeration shows these are the only two store locations that .NET makes available.
