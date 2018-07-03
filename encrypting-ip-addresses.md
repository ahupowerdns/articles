---
title: "Encrypting Ip Addresses"
date: 2017-05-07T20:46:14+02:00
draft: false
---

# On IP address encryption: security analysis with respect for privacy

Frequently, privacy concerns and regulations get in the way of security
analysis. I’m a big fan of privacy, but I’m also a big fan of security and
preventing people from getting hacked. If you are hacked you have no privacy
either.

Per-customer/subscriber traces are extremely useful for researching the security
of networks. Specifically, lists of DNS requests per IP address make spotting
infected users or infected devices very easy, especially if you look at
“yesterday’s traffic” and compare it to today’s threat intelligence.

However, privacy officers rightly object the unbridled sharing of which IP
address visited what website. So what to do? Customers that are hacked surely
have less privacy than ones that aren’t, so there is merit to doing the
research.

#### Encrypting IP addresses

With infinite information, we could do the best analysis. However, infinite
information actually hurts privacy more than it helps. An interesting middle
ground is where we have the information for analysis, but it does not
immediately impact privacy.

One potential solution is to encrypt IP addresses in log files with a secret
key. This key is held by the privacy officer, or their department, and if based
on encrypted IP addresses something interesting is found, the address can be
decrypted for further action.

Before delving into the pros and cons of this, let us look at the technology
involved. To enable existing tooling to remain useful, an IP address must remain
an IP address of the same length or representation. To encrypt a addresses in a
PCAP file for example means leaving all structures in place, but to change the
32 bits of an IPv4 address into 32 other bits.

This is not as simple as it sounds. Conventional well known block ciphers
operate on 56, 64, 128 bits or even more. We could zero-pad a 32 bit IPv4
address of course and encrypt it, but the output would be 128 bits. And all 128
bits would be needed to recover the original 32 bits!

Another solution is to XOR the 32 bit IP address but the problem with that is
that XOR is useless as an encryption algorithm. For example, 1.2.3.4 and 1.2.3.5
would XOR to almost exactly the same value. This is not encryption.

Stream ciphers are not useful since they reveal too much of the structure of IP
addresses. All of 1.2.3.0/24 ends up with the same first 24 bits.

Until recently, the “best” published solution for 32-bit encryption was Skip32,
(a variant of [Skipjack](https://en.wikipedia.org/wiki/Skipjack_(cipher)))
authored by Greg Rose in 1999. Sadly, Skip32 is extremely weak, and can’t be
recommended for use.

#### ipcrypt

In 2015 noted cryptographer [Jean-Philippe Aumasson](https://aumasson.jp/)
released ‘[ipcrypt](https://github.com/veorq/ipcrypt)’, inspired from
[SipHash](https://en.wikipedia.org/wiki/SipHash) (which was invented by Aumasson
and Dan J. Bernstein).

ipcrypt encrypts IPv4 addresses to other IPv4 addresses using a 128 bit key. The
reference implementations are in Go and Python and operate on ASCII versions of
IP addresses.

A [C version](https://github.com/jedisct1/c-ipcrypt) is also available, authored
by noted developer & security researcher [Frank Denis](https://00f.net/). This
version operates on binary representations of IPv4 addresses.

Little formal analysis of ipcrypt is available, but given its heritage and
authorship, it seems like a reliable choice.

> Note that Jean-Philippe Aumasson’s new book “Serious Cryptography” will be
> available in August 2017. Preview and a sample chapter are available over at [No
Starch Press](https://www.nostarch.com/seriouscrypto).

#### IPv6 encryption

IPv6 makes this whole process a lot simpler. Since IPv6 addresses are 128 bits
long, many block ciphers are available for use. AES is widely available and
suitable. No encryption mode is required (“straight ECB”).

#### Pros and cons

Now, in a Twitter discussion on this subject, various people immediately voiced
that encrypting IP addresses is not perfect. Sadly nothing is. Encrypting an IP
address using a static key does not provide anonymity but pseudonymity. Given
sufficient traces and logs, the “personality” of an IP address becomes apparent.

If the DNS traffic of IP address 0.12.24.1 includes a lot of sites relevant to
DNS, DNA, startups, physics, PowerDNS and Open-Xchange, you can be sure that
0.12.24.1 is me.

In addition, as with any encryption system, the static key used for encryption
might of course leak. This would suddenly make all encrypted files ‘the real
deal’ again.

Both worries can be addressed somewhat by frequently changing and even
destroying the key. Using a new key every day for example means that no long
term correlation of data is possible, since a customer IP address encrypts to a
new address after at most 24 hours. Secondly, destroying the key severely limits
the damage that old files can do.

The upside of doing such analysis however is amazing. Over at PowerDNS we
frequently analyse anonymised traces of our customer’s subscribers DNS traffic
and this enables us to immediately spot compromised accounts, which then enables
PowerDNS users to act on this data.

#### Summarising

Privacy regulations make it hard to do all the analysis we want. Encrypting IP
addresses ‘in place’ using a frequently changing key makes it possible do do
such analysis pseudonymously, and still spot problems. Privacy officers can then
decrypt relevant IP addresses to take action.

