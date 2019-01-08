---
published: true
layout: post
title: Service-Specific Certificate Store
draft: true
---

Recently I ran into situation where I needed to access the Certificate Store *scoped* to a specific Windows Service.  If you're not familiar with this, the Certificate Store on Windows can be scoped to different levels.  There's the global store that's scoped to `LocalMachine`, then there are user-specific stores which default to the `CurrentUser`.

Additionally, since at least Windows 2008, Certificate Store access can be scoped to a specific Windows Service.  This can be useful if you want to compartmentalize and isolate certain certificates to certain services.  For some services, this may even enable certain unique features, which is the use case that brought me here.  But accessing the service-specific certificate store in an automation-friendly manner turns out to be a challenge.

(TL;DR -- If you've come here just looking for the tooling to enable this, you can jump right to [ServiceCertStore Tools](#servicecertstore-tools).)

## Accessing Service-specific Stores Manually

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

## Extending .NET Platform Support for Certificate Stores

Fortunately, the [`X509Store` class](https://docs.microsoft.com/en-us/dotnet/api/system.security.cryptography.x509certificates.x509store) provides an escape hatch.  The `X509Store` class is the standard representation and interface into a Windows certificate store in the .NET Base Class Library (BCL).  It provides a number of [constructors](https://docs.microsoft.com/en-us/dotnet/api/system.security.cryptography.x509certificates.x509store.-ctor) that you can use to open up a particular certificate store based on [location](https://docs.microsoft.com/en-us/dotnet/api/system.security.cryptography.x509certificates.storelocation) and [name](https://docs.microsoft.com/en-us/dotnet/api/system.security.cryptography.x509certificates.storename).

But it also provides one additional constructor that takes an `IntPtr` as a handle to an `HCERTSTORE` store:

```csharp
public X509Store (IntPtr storeHandle);
```

Digging around a bit into the Win32 API, we find that the [`CertOpenStore`](https://docs.microsoft.com/en-us/windows/desktop/api/wincrypt/nf-wincrypt-certopenstore) function of the WinCrypt part of the Win32 API returns just such a handle:

```cpp
HCERTSTORE CertOpenStore(
  LPCSTR            lpszStoreProvider,
  DWORD             dwEncodingType,
  HCRYPTPROV_LEGACY hCryptProv,
  DWORD             dwFlags,
  const void        *pvPara
);
```

Further digging reveals that this one function can be used to open certificate stores across all the different store locations and scopes, including [service-specific stores](https://docs.microsoft.com/en-us/windows/desktop/SecCrypto/system-store-locations).  Perfect!

All we need now is a way to access this Win32 API call from a .NET context, which is where the [pinvoke.net](https://www.pinvoke.net/) site comes in very handy.  A quick search on the site, gives us [exactly the code we need](https://www.pinvoke.net/default.aspx/crypt32.certopenstore) to access the API directly from C#:

```csharp
[DllImport("CRYPT32.DLL", EntryPoint="CertOpenStore", CharSet=CharSet.Auto, SetLastError=true)]
public static extern IntPtr CertOpenStoreStringPara( int storeProvider, int encodingType,
   IntPtr hcryptProv, int flags, String pvPara);

[DllImport("CRYPT32.DLL", EntryPoint="CertOpenStore", CharSet=CharSet.Auto, SetLastError=true)]
public static extern IntPtr CertOpenStoreIntPtrPara( int storeProvider, int encodingType,
   IntPtr hcryptProv, int flags, IntPtr pvPara);
```

There are two variations of this API call, corresponding to the two different ways that the function interprets its combination of arguments.  Based on the `CertOpenStore` docs, we know we need to use the first variation that allows us to pass in the name of the service in combination with the store-name in the form `ServiceName\StoreName` as the `pvPara` argument.  Putting this together we're able to invoke the correct form of the function, obtain a handle to the store in the form of an `IntPtr` and pass that into the constructor for `X509Store`:

```csharp
    [DllImport("CRYPT32.DLL", EntryPoint = "CertOpenStore", CharSet = CharSet.Auto, SetLastError = true)]
    internal static extern IntPtr CertOpenStoreStringPara(int storeProvider, int encodingType,
                                                          IntPtr hcryptProv, int flags, String pvPara);

	public static X509Store OpenStore(string serviceName, string storeName)
    {
      var storeHandle = CertOpenStoreStringPara(
        13           // CERT_STORE_PROV_SYSTEM_REGISTRY_W
        ,0           // No encoding type for registry stores
        ,IntPtr.Zero // NULL for the crypto provider implies default
        ,(5 << 16)   // CERT_SYSTEM_STORE_SERVICES_ID for Upper Word
        , $"{serviceName}\\{storeName}");
      
      return new X509Store(storeHandle);
    }
```

Once we have this working code in .NET, we can leverage in PowerShell.  We can't easily integrate into PowerShell's Certificate provider, but we can make various calls into the `X509Store` class to do things like enumerate existing certificates, add new certificates and remove expired certificates.

## ServiceCertStore Tools

I've wrapped up all this basic functionality along with a bit more error handling and a convenient utility class, and made it available as:

* a .NET assembly available on [nuget](https://www.nuget.org/packages/Zyborg.Security.Cryptography.ServiceCertStore/)
* a PowerShell Module that offers the basic list, add and remove features on the [PowerShell Gallery](https://www.powershellgallery.com/packages/ServiceCertStore)
* a sample standalone [CLI app](https://github.com/zyborg/Zyborg.Security.Cryptography.ServiceCertStore/releases/download/v1.0.1.0/servicecerts-cli-1_0_1_0.zip) with similar features

All of the source code is available on [GitHub](https://github.com/zyborg/Zyborg.Security.Cryptography.ServiceCertStore/).

Enjoy!
