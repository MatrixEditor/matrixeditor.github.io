---
title: Dementor Part 1 - Introduction
author: me
date: 2025-11-02 00:00:00
categories: ["Active Directory", "Tools", "Dementor"]
tags: [":ad", "protocols", "tool"]
render_with_liquid: false
description: What is that strange tool 'Dementor' and what does it offer?
image:
    path: /assets/img/posts/ad/2025-11-02-dementor-logo.jpg
---

# **Dementor**{: .dm-title-gradient}
{: data-toc-skip='' }

What is it exactly? To keep it short: Dementor is a rogue service provider capable of poisoning specific protocol requests, such as mDNS or LLMNR. You can think of it as an enhanced version of [*responder*](https://github.com/lgandx/Responder).

I will explore implementation details and core concepts throughout this series about the tool. This post is just about the idea for this tool and why it is still an upgrade.

## Background

The idea, and the need, for *Dementor*{: .dm-title-gradient} came up when I started digging into Active Directory and penetration testing of such environments. Most blog posts showcased *responder* for multicast poisoning; back then reading the sources was like digging through a mess (*no offense intended*).

What I needed was a tool with a much more granular configuration. And some weekends later, *Dementor*{: .dm-title-gradient} was written (or half of it to be precise).

> **Disclaimer**: *Dementor* is intended for educational purposes only. Do *not* run this tool against systems you do not own or for which you do
> not have explicit written permission to test. Misuse may be illegal and is
> solely the user's responsibility (yours).
{: .prompt-info }

> **Yet another disclaimer**: the logo and image for *Dementor* where generated using AI; the code was originally written by hand. Yes, I know - but it aws still fun.

---

## Offers

With the background covered, the obvious questions are: Why use *Dementor*? If it’s similar to *responder*, why switch?

Please note the scope of *Dementor* differs slightly from *responder*:. *Dementor* functions as a **rogue service** provider and multicast poisoner: it serves protocol responses to capture authentication material, but it also provides a broader and more modular platform for custom handlers. To be clear on what this tool offers:

- No reliance on hardcoded or precomputed packets
- Fine-grained, per-protocol configuration
- Near-complete protocol parity with *responder* plus several additional protocols (e.g. IPP, MySQL, X11, UPnP, ...)
- Easy integration of new protocols
- Significant improvements to credential-capture reliability
- There's also a documenation available (*whaat*), with examples (*no way*)

I will get into the specifics for all of these points below.

> A note of respect: *responder* is a great tool, it inspired much of *Dementor*’s design. Big thanks to the responder developers and maintainers.

### I. No hardcoded flows / packets

One mayjor aspect is no need for hardcoded packets again. Take the following (simplified) snippet from *responder*:

```python
# ...
class RPCMap(BaseRequestHandler):
	def handle(self):
		try:
			data = self.request.recv(1024)
			self.request.settimeout(5)
			Challenge = RandomChallenge()
			if data[0:3] == b"\x05\x00\x0b":#Bind Req.
				if FindNTLMOpcode(data) == b"\x01\x00\x00\x00":
					n = NTLMChallenge(NTLMSSPNtServerChallenge=NetworkRecvBufferPython2or3(Challenge))
					n.calculate()
					RPC = RPCNTLMNego(Data=n)
					RPC.calculate()
# ...
```
{: file="Responder: servers/RPC.py"}

We know that in Windows environments everything can change without notice. *Dementor* uses impacket and scapy instead of using hardcoded packet bytes.


### II. Fine-grained configuration

Another point I observed in other tools was the lack of individual configuration. *Dementor* provides per-service configuration: you can control things like server port, server name (SMB), TTL (mDNS), and other protocol-specific response fields.

### III. Near-complete protocol parity

There is a compatibility matrix showing supported protocols and development status relative to *responder*: [Dementor - Compatibility](https://matrixeditor.github.io/dementor/compat.html).

Notable new protocols supported:

- SSDP (*multicast poisoner*)
- MySQL
- X11
- IPP
- UPnP

Protocols supported by *responder* but not included in *Dementor*:

- DHCP and DNS (*poisoner*) \*
- SNMP
- RDP \*
- HTTP_PROXY \*
- MQTT

> All protocols marked with a `*` **won't** be supported by *Dementor* *unless* a pull request is filed.
> For certain services like DNS, DHCP, support is intentionally omitted because they are risky to abuse in an active network and can cause severe disruption if used incorrectly.
> Alternative specialized tools exist for those protocols; see the compatibility page for details/links.
{: .prompt-info }

### IV. Protocol extensions

Yes, that's right, *Dementor*{: .dm-title-gradient} supports *custom* protocols written by **you**. Adding a new protocol is as simple as referencing your Python handler in the configuration file. *Follow up post would be linked here...*

### V. Capture reliability improvements

Because *responder* historically used hardcoded patterns, it sometimes fails to capture credentials from newer or slightly different client implementations. *Dementor* improves capture reliability by parsing flows dynamically using already existing packet parsers. For example: *responder* does **not** distinguish between an NTLMv1/2 and NTLMv1/2-SSP hash when capturing credentials with the MSRPC server.

### VI. Documentation

Yes - there is documentation. Full configuration options and examples are available online: [Dementor - Docs](https://matrixeditor.github.io/dementor/).

## Next Steps

The next post in this series will dive into *Dementor*’s core architecture and explain the design decisions made during development. See ...