Ford SYNC 3 is unable to connect to secured 802.11r wireless networks
===

Ford SYNC 3 includes an 802.11b/g/n wifi client, but it is unable to connecting to wireless networks that are a) secured using WPA2 and b) support 802.11r Fast Transitions. This is an implementation defect and a clear violation of the 802.11 standard.

Background
---

WPA2 is the standard mechanism for securing 802.11 wireless networks.

802.11r Fast Transitions are a feature intended to support wireless clients that require mobility and, well, fast transitions between access points. Devices such as VoIP phones benefit from the ability to transition from one access point to another with minimal disruption to traffic flow. 802.11r FT is an optional feature incorporated into 802.11n and into the mainline standard as of IEEE Std 802.11-2012.

Issue
---

When SYNC 3's 802.11n client software attempts to connect to an access point that supports 802.11r Fast Transitions, it attempts an 802.11r FT association using a **non-802.11r** security scheme. This is expressly prohibited; the standard requires that access points refuse such connections. As a consequence, SYNC 3 is unable to connect.

Technical description
---

SYNC 3 constructs and transmits an Association Request frame with two contradictory pieces of information in this scenario.

First is the Mobility Domain element (MDE). This element indicates that SYNC 3 is a) aware of 802.11r and b) wishes to connect to a specific 802.11r mobility domain. The inclusion of an MDE makes SYNC 3's association request an FT initial mobility domain association in an RSN per IEEE Std 802.11-2012 § 12.4.2:

> A STA indicates its support for the FT procedures by including the MDE in the (Re)Association Request frame and indicates its support of security by including the RSNE.

Second is the Robust Securtity Network element (RSNE) which describes the parameters of the security scheme SYNC 3 will use. This is part of a negotiation with the access point; the access point states what it supports using RSNEs in Beacon and Probe Response frames, and the wireless client bases its RSNE on the intersection of what the AP supports and what the client supports. The RSNE includes an Authentication and Key Management (AKM) Suite selector, which dictates the procedure used to derive encryption keys.

An 802.11r-capable access point configured for WPA2-PSK will announce support for two AKM suites: `00-0F-AC:2` PSK and `00-0F-AC:4` FT authentication using PSK. (See IEEE Std 802.11-2012 § 8.4.2.27.3 for a table.) FT authentication is designed to support transitions between multiple access points, whereas non-FT authentication is not.

SYNC 3's association request includes an MDE (indicating a desire to use FT) and a RSNE listing AKM suite `00-0F-AC:2` (indicating a desire to use non-FT authentication). This is expressly prohibited per IEEE Std 802.11-2012 § 12.4.2:

> If an MDE is present in the (Re)Association Request frame and the contents of the RSNE do not indicate a negotiated AKM of Fast BSS Transition (suite type `00-0F-AC:3`, `00-0F-AC:4`, or `00-0F-AC:9`), the AP shall reject the (Re)Association Request frame with status code 43 (i.e., Invalid AKMP).

It also doesn't make any objective sense, because fast transitions simply depend on FT authentication. 

Solutions
---

SYNC 3 must either:

1. Stop attempting 802.11r FT associations (i.e. remove the MDE from its association request), or
2. Begin attempting 802.11r FT associations using 802.11r FT authentication (i.e. update the WPA supplicant to support 802.11r AKM suites).

Progress
--

Ford is tracking this issue as `CAS-9606059`, a research case opened 2016-05-31. I also reported this issue using several other channels earlier in the month of May, but none of those Ford representatives were able to give me any identifier.

On June 6, a Senior Business Analyst at Ford contacted me asking me to try disabling authentication or to use a different password. I informed her that I had already performed much experimentation, and that I could send two packet captures – one showing SYNC 3 connecting to an access point successfully, and another showing SYNC 3 failing to connect to that same access point once 802.11r is enabled. She allowed me to forward these packet captures by email. I also included a [dissection of one problematic association request](http://i.imgur.com/Kap11SZ.png), showing that SYNC 3 selected a non-FT AKM and included a mobility domain element in violation of the 802.11 standard.

On July 14, a different analyst contacted me to say that the issue was still open, and to ask if I could send over those packet captures. I forwarded the email from June, which she acknowledged but which her reply only partially quoted. I replied again attaching a PDF of my June email, which she acknowledged again.

Over the next several months, I received periodic phone calls from Michigan telling me that the case is still open and that there's no ETA. On November 15, I emailed the second analyst again asking for a status update.
