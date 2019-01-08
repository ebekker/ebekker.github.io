---
published: true
draft: true
title: ACMESharp is Dead.  Long Live ACMESharp!
author: Eugene Bekker
---
It's a bit overdue, but I've begun work on ACMESharp 2.0 or what I'm branding as ACMESharpCore.
The goal of this version is to address a few shortcomings and changes that are either necessary
or desired.  Some folks have questioned if the ACMESharp project is still alive and I wanted to
let people know that the project *is* alive, but also what they can expect and when.

# How ACMESharp Came to Be

The ACMESharp project officially began in 2015.  After I discovered the Let's Encrypt effort, about
a year before that, I had initially [proposed](https://github.com/certbot/certbot/issues/116) that LE
would itself include support for the Windows platform and in a Windows-centric manner.  But they were
focused on just getting the basic infrastructure off the ground and their initial focus was on Linux
which is understandable -- you have to walk before you can run.

Initially I was just toying around with some of the underlying concepts.  After reviewing the
[initial specification](https://github.com/letsencrypt/acme-spec) of ACME -- the protocol that
facilitates the automated and self-service nature of the Let's Encrypt CA, I started to experiment
with just a few of the key concepts and make sure that it would be doable.  But it would take about
another year before I actually got serious about it and kicked off a real project, which was
initially called `letsencrypt-win`.  (Before that, I was actually hoping and waiting for someone
else to kick off the Windows support...)

The motivation came from my own self-interest -- I wanted to leverage the LE certs for personal
projects as well as in my professional work and so when an opportunity arose to dedicate some time
and justify it, I jumped at it.  Now, I will be the first to admit that I have not been the ideal
steward for this OSS project.  If you look at the
[overall contribution graph](https://github.com/ebekker/ACMESharp/graphs/contributors) you'll see
that it's full of lots of periods of downtime and just a few *peaks of inspiration*.  But, I am
glad that this personal pet project of mine has been able to
[help and even inspire](https://github.com/ebekker/ACMESharp/wiki/Contributions) others to scratch
their own itch.

One of the things that I'm most proud of and most grateful for is that a little community has
developed around the project and especially a few key folks who have taken it upon themselves
to help others by answering questions in issues, or writing supplementary scripts to offset the
shortcomings of the core project, or contributing a Provider library to extend native support
for a new service provider -- thank you all, it's definitely been noticed and appreciated!

# Where ACMESharp is Going

So now that it's not quite three years going, the project is now (over)due for a little refresh.
The .NET platform and related technologies like PowerShell have gone through a bit of a
renaissance in the mean time.  The Let's Encrypt project has matured, released and been
a big success, and ACMESharp users have been providing valuable feedback all the while.

So I've found a bit of time to squeeze in some needed attention to ACMESharp and here are the
major things that I've been thinking about and meaning to do for a while now, and finally
underway:

* **ACMESharp .NET projects are moving to .NET Standard and .NET Core.**
* **Protocol support is moving to the latest ACME specification (ACME v2).**
* **Support will be added for the pieces of the protocol that were missing in the past.**
* **Restructuring the project.**
* **Moving to the MIT license.**

In the rest of this post, I'll talk a bit about each of these and my thoughts.

## Moving to .NET Standard and .NET Core

The .NET platform has evolved a good bit in the last few years with the move to OSS,
adoption of simpler and cleaner
[app models](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel?view=aspnetcore-2.1&tabs=aspnetcore2x),
[cross-platform support](https://github.com/dotnet/core/blob/master/release-notes/download-archives/2.1.0-preview2-download.md),
a focus on [performance](https://blogs.msdn.microsoft.com/dotnet/2018/04/18/performance-improvements-in-net-core-2-1/)
that is stronger than ever and exciting new features like
[AOT](https://github.com/dotnet/corert),
[WASM](https://blazor.net/) and 
[advanced memory access patterns](https://msdn.microsoft.com/en-us/magazine/mt814808.aspx),
it has never been more exciting or more empowering to be a .NET developer.

To take advantage of many of these advancements, ACMESharp will move to .NET Standard (and
.NET Core where appropriate).  Moving to .NET Standard will allow the core client library
to be usable across the various different .NET runtimes, including the classic .NET Framework
as well as .NET Core and Mono where all the exciting new work is being done.

In some cases we'll be able to leverage new
[out-of-the-box features](https://github.com/dotnet/corefx/issues/17892) to reduce our
dependencies on external libraries, but as some of these have not been absorbed into the .NET
Standard itself, these pieces may need to be contained within a .NET Core-specific module.

As before, support for a Provider model will allow extensibility in various areas of the library
but there will be a stronger delineation between the protocol core that implements ACME and the
supporting and related areas like Authorization support.  I'll touch on that in more detail down
below when I talk about restructuring.

## Moving to ACME v2

The Let's Encrypt project has had time over the last few years and a
[few million certificates](https://letsencrypt.org/stats/#growth) to test out their initial
protocol specification -- what is now commonly referred to as ACME v1.  From those learnings
they have refined the specification and reshaped it into what will eventually become an
[Internet Standard](https://tools.ietf.org/html/draft-ietf-acme-acme-12) as ACME v2.

The general flow of how ACME works still remains the same, but some of the underlying details
and mechanics are now very different.  With ACMESharpCore, we're moving to full ACME v2 support
and dropping ACME v1.  The old code base will still work for any ACME v1 endpoint, but as full
support for v2 has been [released](https://letsencrypt.org/2017/06/14/acme-v2-api.html) and
will be the path moving forward, focusing only on v2 will allow faster development and a cleaner
design for the next generation of ACMESharp.

## Missing Features Will Be Implemented

Now, some parts of ACME v1 were never implemented or supported in ACMESharp.  Things like
revocation or recovery keys as base protocol operations, or support for TLS-based challenges
were never completed due to lack of time and because they were not strictly needed to implement
the core functionality of obtaining certificates.

In some cases these turned out to be a good decisions as some of these features were never
carried over to the newer v2 protocol spec.  For example, TLS support (TLS-SNI) for Authorization
Challenges is not currently supported or defined in ACME v2 because of a better understanding
of possible threat models and vulnerabilities that may be encountered.

But other features really needed to be implemented to provide more complete and robust support
for ACME interoperability.

In ACMESharpCore some of these missing features will be addressed from the very start along with
support for new features introduced in ACME v2.  Some examples include:

* Account Revocation and Key Roll-Over
* Authorization Revocation
* Elliptic Curve support for Account Keys as well as Certificates

These are just a few of the things in the core protocol client library, but I hope to make
similar improvements in the supporting libraries and tools, which leads me to the changes in
project structure.

## Restructuring the Project

After working with the existing project structure for a few years, I found a few issues with
the way that the project was initially setup and hindered its growth and community support.
For starters, as a GitHub project directly under my personal user account, it prevented me
from allowing outside contributors from becoming first-class members of the project and
managing contributions with the same level of authority.

### PKISharp - GitHub Organization

To address this challenge a separate project [organization](https://github.com/PKISharp) has
been setup and will serve as the new home for ACMESharpCore and related projects, such as other
PKI tools that are complementary to ACME-based certificate management.  A little while back, the
[documentation](https://pkisharp.github.io/ACMESharp-docs/) for the original ACMESharp code
base was revamped and it now lives
[under this PKISharp org](https://github.com/PKISharp/ACMESharp-docs) and more recently the
maintainers of the [win-acme](https://github.com/PKISharp/win-acme) project
(formerly letsencrypt-win-simple) -- a standalone client for simplified IIS certificate
setup using ACME certs -- moved under this org as well.

[ACMESharpCore](https://github.com/PKISharp/ACMESharpCore) began life under this org from
the start, initially as just a playground to experiment with some new ideas, and now the
the home of the official ACMESharp 2.x code base.

### Reorganizing Solutions and Projects

Another area that I think caused confusion and created obstacles was the internal
structure of the projects themselves.  For starters, right from the beginning there was a
single solution that housed all of the client library code, the different Provider
implementations and the PowerShell modules.  This mixed things up in various ways because
it blended problem reports into a single issues list, documentation across the different
components was combined in an awkward way since each component was really serving a
different audience (i.e. client library for developers, and PowerShell modules for operators)
and I was hesitant to accept certain contributions because of giving the wrong impression
that some externally contributed code was *officially* supported or tested with the core
project.

Going forward, there is going to be a much stronger separation of components.
For starters, the core client library will be broken out into its own GitHub project and
this will mean the code base will be smaller and more focused, which will mean that someone
who wants to learn about the inner workings and contribute should have an easier time.
Since the issue list will be also be dedicated to just the client library, hopefully
this will make it easer to find or report problems that are more relevant to the specific
code base.

The PowerShell modules will live in their own GitHub repository and again this will allow
issues specific to this client implementation to be easier to search and report.  Speaking
of PowerShell, the new 2.x version of the modules will be implemented as
[PowerShell Standard](https://github.com/PowerShell/PowerShellStandard) modules which means
they will work with either classic Windows PowerShell or the new
[PowerShell Core](https://github.com/PowerShell/PowerShell) (v6).  For those of you who
work with PWSH on a regular basis and are now starting to support and manage environments
other than Windows, this will allow you to manage ACME certs with a single scripting and
automation platform that runs across all the different target environments.

Finally, separate *external contributions* projects will be setup that will house any
external Provider implementations that are not *officially* supported but can be useful
to the community.  Typically these are extensions that target service providers that
cannot be easily tested or supported without some financial commitment or due to some
other technical challenge.  We don't want to turn away any contributions, but sometimes
it's just not feasible to support some targets.  In the past, I've asked contributors to
host these in their own repos and the ACMESharp project page would just reference link to
these repos, but that hasn't worked out for various reasons, so hopefully adding these
will reduce the barriers and improve contributor engagement.

## Moving to MIT License

Lastly, one of the changes with ACMESharpCore will be the adoption of the MIT license.
This seems to be the *norm* for the .NET OSS community spurred by Microsoft's own decisions
and hopefully this encourages better community and usability.  I don't forsee any problems
as this becomes less restrictive than the original code base and any contributions that
may have been attributed to authors under the original license and carried forward will
be squared away as needed.

# So What's Next?

I'm glad you asked!   So the good news is, a good bit of the work is already underway
and even completed.  The
[ACMESharpCore client library](https://github.com/PKISharp/ACMESharpCore) repo already
has working code, including integration tests that already implements a number of new
features not found in the original code base, here are just a few highlights:

* Supports creating/updating/revoking Accounts
* Supports creating/updating/revoking Orders/Authorizations
* Supports RSA-based and EC-based Account keys
* Supports RSA-based and EC-based Certificate keys
* Implemented as .NET Core App 2.0
* Using async code base throughout

A much improved testing infrastructure has been put in place to support integration
testing with the LE staging endpoints -- this will allow better testing and easier
to support testing going forward.  The tests are actually split out cleanly between
**unit tests** based on [MSTest v2](https://github.com/Microsoft/testfx) and
**integration tests** based on [xUnit](https://xunit.github.io/).  While my personal
preference is for MSTest, xUnit provides a better base platform to extend for
features needed specifically for integration testing, which is why there are two
testing frameworks in use.

The next step is to get documentation and the CI/CD infrastructure in place and that
will enable automated publishing of artifacts which will make the new code easily
accessible to developers and operators. Stay tuned...
