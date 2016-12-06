Ford SYNC 3 is unable to connect to secured 802.11r wireless networks
===

Ford SYNC 3 includes an 802.11b/g/n wifi client, but it is unable to connecting to wireless networks that a) are secured using WPA2 and b) support 802.11r Fast Transitions. This is an implementation defect and a clear violation of the 802.11 standard.

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

First is the Mobility Domain element (MDE). This element indicates that SYNC 3 a) is aware of 802.11r and b) wishes to connect to a specific 802.11r mobility domain. The inclusion of an MDE makes SYNC 3's association request an "FT initial mobility domain association in an RSN" per IEEE Std 802.11-2012 § 12.4.2:

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

Over the next several months, I received periodic phone calls from Michigan telling me that the case is still open and that there's no ETA.

On November 15, I emailed the second analyst again asking for a status update. A brief discussion ensued, and I pointed out that I had completed research into this bug and provided my technical findings to my dealership within 72 hours of taking delivery of my vehicle, and that Ford appears to have sat on it for six months and made no progress. On November 16, I was told that the issue had been sent to CCT Escalation and that Engineering remained in the loop. This escalation is identified as `CAS-9606059-T9J7D6 CRM:00013000000371`.

On November 17, I was contacted by a Ford Regional Customer Service Manager who informed me that she would be happy to assit with her technical resources… once I dropped off the vehicle at a Ford dealership. I replied that I understood she had a protocol to follow, but that it makes little sense for me to drop off my vehicle given the nature of the problem -- that instead, someone at Ford could read my report, look at their software, find and fix the problem without requiring my car. I also added that I would be willing to drop off my car anyway at any of the six closest dealerships anyway, so long as a comparable loaner was available.

Also on November 17, it occurred to me that Wireshark (the protocol analyzer of record) can and should identify this issue automatically. I [opened a bug](https://bugs.wireshark.org/bugzilla/show_bug.cgi?id=13149) describing the feature I think it should have, [submitted a patch](https://code.wireshark.org/review/#/c/18862/), and [got it merged 🎉](https://github.com/wireshark/wireshark/commit/50515b9ebf8db7e97369e0cdbc748db9d0fd818b). The association request frames [now look like this](http://i.imgur.com/ZA2i5Uh.png), making this issue obvious without needing to squint at IEEE standards.

Over the next several days, I had a back-and-forth with the Ford Regional Customer Service Manager about how to proceed with this case. She needs me to go to a dealership and do _something_ to demonstrate the issue. I explained that bringing my network with me would be an ordeal, and asked if I could bring electronic evidence instead -- say, a video of me using my car trying to connect to my network along with packet captures of the exchange. She seemed to think that would work, but I went a step further.

Ultimately, all I need to demonstrate the issue is an 802.11r-capable access point, so I ordered up a [$30 router](https://www.amazon.com/gp/product/B01DBS5Z0W). It arrived on November 25. I then [flashed it with a stock OpenWRT image](https://wiki.openwrt.org/toh/gl-inet/gl-mt300a#oem_easy_installation), [configured it for 802.11r](https://www.reddit.com/r/openwrt/comments/515oea/finally_got_80211r_roaming_working/), and went out to my car with the box and `iwcap`… only to find that `hostapd` doesn't actually validate the AKM suite as required by the 802.11 standard. So, I fixed `hostapd`, built and installed a [replacement `wpad` package](https://s3.amazonaws.com/willglynn/hostapd/wpad_2016-01-15-2_ramips.ipk), verified that it now corrrectly refuses the broken association request, and [submitted my patch](http://lists.infradead.org/pipermail/hostap/2016-November/036706.html) (which was [subsequently merged 🎉](http://w1.fi/cgit/hostap/commit/?id=209dad066e5275ac13f52623cc9eaf9b70910123)). I then told my contact at Ford that I have a [portable 802.11r lab](http://i.imgur.com/OD9RYhj.jpg) that I can bring to the dealership of her choosing.

On November 28, I learned that the Regional Customer Service Manager that took this case on November 17 was changing positions and that my case would be handled by yet another representative, now my fourth contact at Ford corporate. (What kind of turnover do they have? Is this normal?)

On November 30, I brought my vehicle to a dealership per Ford's request, where they identify this as repair order number `6220998`. The dealership was not interested in my portable 802.11r lab, but the technician at the dealership's service center was able to confirm the issue using their internal wireless network, which apparently is also 802.11r-capable. They referred the case to some internal Ford hotline.

On December 5, I received an email from the second Regional Customer Service Manager starting with:

> At this time it has been determined that the cause of the concern is the customer’s connection point and not a vehicle failure. There has been no diagnosis and therefore no repair remedy to apply in this case. As I am not a technical resource, I am unable to make any recommendations, but I must inform you that I have exhausted my technical resources and have no other recourse to follow.

I replied:

> This is not satisfactory.
> 
> I understand that it's convenient to blame the customer's equipment, but I have two different access points from two different vendors both showing that this vehicle is in violation of the IEEE 802.11 standard. Both access points can document their communications with my vehicle, and both show the vehicle sending an association request frame which the 802.11 standard says **the access point must reject**. Please explain how you conclude that this is a problem with my equipment and not a problem with the vehicle.

Another back and forth ensued, and the Regional Customer Service Manager informed me that the dealership had not requested that this case be reviewed by a Field Service Engineer, and that she could work with them to make that happen. I insisted that the engineer be presented with the information contained in this document, saying in part:

> I want to know that some technical resource at Ford has actually **looked** at this information. If they did, I guarantee they would have more to say than "the cause of the concern is the customer’s connection point".
