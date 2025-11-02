---
title: Dementor Part 1 - Introduction
author: me
date: 2025-11-02 00:00:00
categories: ["Active Directory", "Tools", "Dementor"]
tags: [":ad", "protocols", "tool"]
render_with_liquid: false
description: What is that strange tool called 'Dementor' and what does it offer?
image:
    path: /assets/img/posts/ad/2025-11-02-dementor-logo.jpg
---

# **Dementor**{: .dm-title-gradient}
{: data-toc-skip='' }

What is it exactly? To keep it short: Dementor is a rogue service provider capable of poisoning specific protocol requests, such as mDNS or LLMNR. You can think of it as an enhanced, feature-rich successor to the well-known [*responder*](https://github.com/lgandx/Responder).

I will explore implementation details and core concepts throughout this series about the tool. In this post I describe how the idea for this tool came up and briefly show the benefits you can expect when using it.

## Background

The idea — and the need — for *Dementor*{: .dm-title-gradient} arose when I started focusing on Active Directory security and penetration testing. Most blog posts I read showcased *responder* for multicast poisoning; I wanted to understand the technique in-depth, so my first step was to read the source code - it was/is a mess. The codebase relied heavily on hardcoded communication sequences and lacked a clear, consistent structure (*no offense intended to the original authors*).

Further scrolling through the source code of *responder* led to the idea of a revised tool: without hardcoded flows and with much more granular configuration.
From that idea *Dementor*{: .dm-title-gradient} emerged from the shadows. To be exact, it was formed and designed outside the matrix.

> **Disclaimer**: *Dementor* is intended for educational purposes only. Do *not* run this tool against systems you do not own or for which you do
> not have explicit written permission to test. Misuse may be illegal and is
> solely the user's responsibility (yours).
{: .prompt-info }

---

## Offers

With the background covered, the obvious questions are: Why use *Dementor*? If it’s similar to *responder*, why switch? Those are valid; here’s why *Dementor*{: .dm-title-gradient} is worth considering as an alternative, and where responder continues to be valuable.

Please note the scope of *Dementor* differs slightly from *responder*, but the two overlap significantly. *Dementor* functions as a **rogue service** provider and multicast poisoner: it serves protocol responses to capture authentication material, but it also provides a broader and more modular platform for custom handlers. To be clear on what this tool offers:

- No reliance on hardcoded or precomputed packets
- Fine-grained, per-protocol configuration via a modular extension system (*this is huge*)
- Near-complete protocol parity with *responder* plus several additional protocols (e.g. IPP, MySQL, X11, UPnP, ...)
- Easy integration of new protocols via the extension system
- Significant improvements to credential-capture reliability (*this is also huge*)
- There's also a documenation available (*whaat*), with examples (*no way*)

I will get into the specifics for all of these points below.

> A note of respect: *responder* is a great tool — it inspired much of *Dementor*’s design. Big thanks to the responder developers and maintainers.

### I. No hardcoded flows / packets

A major advantage of *Dementor* is that it does not depend on hardcoded packet blobs. Hardcoded flows make the codebase hard to audit and easier for detection mechanisms to flag. For example, the following (simplified) snippet from *responder* demonstrates how a server handler builds responses with assumptions about incoming data:

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

This approach often assumes data is always at the expected offsets and formats, which reduces robustness.

*Dementor* addresses this by using (mostly impacket and scapy) third-party libraries to parse incoming packets and maintain flow context. Responses are constructed programmatically based on observed data and the protocol state, making the behavior easier to follow in code and more resilient against client variations.

### II. Fine-grained configuration

One shortcoming I observed in other tools was the inability to tune individual service responses. *Dementor* provides per-service configuration — you can control things like server port, server name (SMB), TTL (mDNS), and other protocol-specific response fields. While *responder* exposes many useful options, configuration for non-HTTP services and the returned payload content is limited. I designed *Dementor* to close that gap and provide you more precise control where it matters.

### III. Near-complete protocol parity

There is a compatibility matrix showing supported protocols and development status relative to *responder*: [Dementor - Compatibility](https://matrixeditor.github.io/dementor/compat.html). The comparison shows that protocol coverage is largely equivalent, and in some cases broader.

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
> For certain services (DNS, DHCP), support is intentionally omitted because they are risky to operate in an active network and can cause severe disruption if used incorrectly.
> Alternative specialized tools exist for those protocols; see the compatibility page for references.
{: .prompt-info }

### IV. Protocol extensions

Yes, that's right, *Dementor*{: .dm-title-gradient} supports *custom* protocols written by **you**. Adding a new protocol is as simple as referencing your Python handler in the configuration file. I plan to publish a follow-up post explaining how to author and integrate a new protocol PoC.

### V. Capture reliability improvements

Because *responder* historically used rigid patterns, it sometimes fails to capture credentials from newer or slightly different client implementations. *Dementor* improves capture reliability by parsing flows dynamically using already existing packet parsers. For instance, *responder* does **not** distinguish between an NTLMv1/2 and NTLMv1/2-SSP hash when capturing credentials with the MSRPC server.

### VI. Documentation

Yes — there is documentation. Full configuration options and examples are available online: [Dementor - Docs](https://matrixeditor.github.io/dementor/).


## Goals

*Dementor*'s primary goal is to provide a clearer, more maintainable platform for learning about specific protocol attacks and demonstrating their impact. It was originally developed as an educational project to deepen my understanding of services and authentication mechanisms commonly used in Windows environments.


## Next Steps

The next post in this series will dive into *Dementor*’s core architecture and explain the design decisions made during development. Part two is coming soon.