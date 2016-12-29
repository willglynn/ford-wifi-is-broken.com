<div class="intro" markdown="1">

Ford SYNC 3 includes an 802.11b/g/n wifi client, but it is unable to connect to wireless networks that support 802.11r Fast Transitions.

When SYNC 3 tries to connect to a network that supports 802.11r, it says "I want to use 802.11r" and "I want to use **non-802.11r** security". That's not allowed, so it can't connect.

That's it.

</div>

Background
---

802.11r Fast Transitions are a feature intended to support wireless clients that require mobility and, well, fast transitions between access points. Devices such as VoIP phones benefit from the ability to transition from one access point to another with minimal disruption to traffic flow. 802.11r FT is an optional feature incorporated into 802.11n and into the mainline standard as of IEEE Std 802.11-2012.

Issue
---

When SYNC 3's 802.11n client software attempts to connect to an access point that supports 802.11r Fast Transitions, it attempts an 802.11r FT association using a **non-802.11r** security scheme. This is expressly prohibited; the standard requires that access points refuse such connections. As a consequence, SYNC 3 is unable to connect.

Technical description
---

SYNC 3 constructs and transmits an Association Request frame with two contradictory pieces of information in this scenario.

First is the Mobility Domain element (MDE). This element indicates that SYNC 3 a) is aware of 802.11r and b) wishes to connect to a specific 802.11r mobility domain. The inclusion of an MDE makes SYNC 3's association request an "FT initial mobility domain association in an RSN" [per IEEE Std 802.11-2012 § 12.4.2](/assets/resources/80211-2012_page_1311.pdf).

Second is the Robust Securtity Network element (RSNE) which describes the parameters of the security scheme SYNC 3 will use. This is part of a negotiation with the access point; the access point states what it supports using RSNEs in Beacon and Probe Response frames, and the wireless client bases its RSNE on the intersection of what the AP supports and what the client supports. The RSNE includes an Authentication and Key Management (AKM) Suite selector, which dictates the procedure used to derive encryption keys.

An 802.11r-capable access point configured for WPA2-PSK will announce support for two AKM suites: `00-0F-AC:2` PSK and `00-0F-AC:4` FT authentication using PSK. (See [IEEE Std 802.11-2012 § 8.4.2.27.3](/assets/resources/80211-2012_page_558_559.pdf) for a table.) FT authentication is designed to support transitions between multiple access points, whereas non-FT authentication is not.

SYNC 3's association request includes an MDE (indicating a desire to use FT) and a RSNE listing AKM suite `00-0F-AC:2` (indicating a desire to use non-FT authentication). This is expressly prohibited [per IEEE Std 802.11-2012 § 12.4.2](/assets/resources/80211-2012_page_1312.pdf):

> If an MDE is present in the (Re)Association Request frame and the contents of the RSNE do not indicate a negotiated AKM of Fast BSS Transition (suite type `00-0F-AC:3`, `00-0F-AC:4`, or `00-0F-AC:9`), the AP shall reject the (Re)Association Request frame with status code 43 (i.e., Invalid AKMP).

It also doesn't make any objective sense, because fast transitions simply depend on FT authentication. 

Solutions
---

SYNC 3 must either:

1. Stop attempting 802.11r FT associations (i.e. remove the MDE from its association request), or
2. Begin attempting 802.11r FT associations using 802.11r FT authentication (i.e. update the WPA supplicant to support 802.11r AKM suites).

Resources
--

I was troubleshooting this on May 4, during which I took two packet captures:

* [one with 802.11r enabled](/assets/resources/may_4_80211r_enabled.pcap.gz), where SYNC 3 can't connect, and
* [another with 802.11r disabled](/assets/resources/may_4_80211r_disabled.pcap.gz), where SYNC 3 can connect.

These were recorded minutes apart using the same Ford vehicle and the same Ruckus access point. The only change was whether the access point advertised its 802.11r capabilities.

Inspecting the 802.11r-enabled capture shows [this association request](/assets/images/association-request-frame.png):

<a href="/assets/images/association-request-frame.png"><img src="/assets/images/association-request-frame.png"></a>

Such an association frame must be rejected by the access point; see [IEEE Std 802.11r-2008 page 54](/assets/resources/80211r-2008_page_54.pdf) or [IEEE Std 802.11-2012 page 1312](/assets/resources/80211-2012_page_1312.pdf).

Software Analysis
--

The theory [which I've long advanced](/2016/11/17/email-1479396709.html) is that there's two components inside SYNC 3, one of which is attempting 802.11r when it sends the association request frame, and another of which is not attempting 802.11r when it handles authentication.

Someone [posted SYNC 3's 2.2 update](http://www.2gfusions.net/showthread.php?tid=3881&pid=100870) which gave me a chance to find out. That archive contains `HN1T-14G381-LG.tar.gz`, which contains `apps.tar.gz`, which contains a QNX6 filesystem holding some software.

This software image confirms at least half my theory. SYNC 3 uses a QNX build of [the usual `wpa_supplicant`](https://w1.fi/wpa_supplicant/) for its 802.11 authentication exchanges. `wpa_supplicant` can support 802.11r, but it must be compiled with `CONFIG_IEEE80211R` and it must be configured as `key_mgmt=PSK FT-PSK`. The default is `key_mgmt=PSK`.

`/apps/NET_WifiConnectionMgr` is a Ford tool which, among other things, creates the configuration file for `wpa_supplicant`. It writes a `network={` block, and it can specify values for:

* `ssid`,
* `bssid`,
* `scan_ssid`,
* `psk`,
* `key_mgmt`, and
* `wep_key0`

However, besides the default of `key_mgmt=PSK`, the only possible key management configuration is `key_mgmt=NONE`.

The generated configuration files **never** contain `key_mgmt=FT-PSK`, meaning that `wpa_supplicant` can **never** attempt FT authentication. This is why the AKM suites in the association request contain only `00-0F-AC:2` (which is `key_mgmt=PSK`) and not `00-0F-AC:4` (which is `key_mgmt=FT-PSK`).

Progress
===

Ford is tracking this issue as `CAS-9606059`, a research case opened 2016-05-31. I also reported this issue using several other channels earlier in the month of May, but none of those Ford representatives were able to give me any identifier.

On June 6, a Senior Business Analyst at Ford contacted me asking me to try disabling authentication or to use a different password. I informed her that I had already performed much experimentation, and that I could send two packet captures – one showing SYNC 3 connecting to an access point successfully, and another showing SYNC 3 failing to connect to that same access point once 802.11r is enabled. She allowed me to forward these packet captures by email. I also included a [dissection of one problematic association request](/assets/images/june-6-dissection.png), showing that SYNC 3 selected a non-FT AKM and included a mobility domain element in violation of the 802.11 standard.

On July 14, a different analyst contacted me to say that the issue was still open, and to ask if I could send over those packet captures. I forwarded the email from June, which she acknowledged but which her reply only partially quoted. I replied again attaching a PDF of my June email, which she acknowledged again.

Over the next several months, I received periodic phone calls from Michigan telling me that the case is still open and that there's no ETA.

Escalation
---

On November 15, I emailed the second analyst again asking for a status update. A brief discussion ensued, and I pointed out that I had completed research into this bug and provided my technical findings to my dealership within 72 hours of taking delivery of my vehicle, and that Ford appears to have sat on it for six months and made no progress. On November 16, I was told that the issue had been sent to CCT Escalation and that Engineering remained in the loop. This escalation is identified as `CAS-9606059-T9J7D6 CRM:00013000000371`.

On November 17, [I was contacted](/2016/11/17/email-1479392945.html) by a Ford Regional Customer Service Manager who informed me that she would be happy to assit with her technical resources… once I dropped off the vehicle at a Ford dealership. [I replied](/2016/11/17/email-1479396709.html) that I understood she had a protocol to follow, but that it makes little sense for me to drop off my vehicle given the nature of the problem -- that instead, someone at Ford could read my report, look at their software, find and fix the problem without requiring my car. I also added that I would be willing to drop off my car anyway at any of the six closest dealerships anyway, so long as a comparable loaner was available.

<aside><p markdown="1">
  Also on November 17, it occurred to me that Wireshark (the protocol analyzer of record) can and should identify this issue automatically. I [opened a bug](https://bugs.wireshark.org/bugzilla/show_bug.cgi?id=13149) describing the feature I think it should have, [submitted a patch](https://code.wireshark.org/review/#/c/18862/), and [got it merged 🎉](https://github.com/wireshark/wireshark/commit/50515b9ebf8db7e97369e0cdbc748db9d0fd818b). The association request frames [now look like this](http://i.imgur.com/ZA2i5Uh.png), making this issue obvious without needing to squint at IEEE standards.
</p></aside>

Over the next several days, I had a back-and-forth with the Ford Regional Customer Service Manager about how to proceed with this case. She needs me to go to a dealership and do _something_ to demonstrate the issue. I explained that bringing my network with me would be an ordeal, and asked if I could bring electronic evidence instead -- say, a video of me using my car trying to connect to my network along with packet captures of the exchange. She seemed to think that would work, but I went a step further.

<aside><p markdown="1">
Ultimately, all I need to demonstrate the issue is an 802.11r-capable access point, so I ordered up a [$30 router](https://www.amazon.com/gp/product/B01DBS5Z0W). It arrived on November 25. I then [flashed it with a stock OpenWRT image](https://wiki.openwrt.org/toh/gl-inet/gl-mt300a#oem_easy_installation), [configured it for 802.11r](https://www.reddit.com/r/openwrt/comments/515oea/finally_got_80211r_roaming_working/), and went out to my car with the box and `iwcap`… only to find that `hostapd` doesn't actually validate the AKM suite as required by the 802.11 standard. So, I fixed `hostapd`, built and installed a [replacement `wpad` package](/assets/resources/wpad_2016-01-15-2_ramips.ipk), verified that it now corrrectly refuses the broken association request, and [submitted my patch](http://lists.infradead.org/pipermail/hostap/2016-November/036706.html) (which was [subsequently merged 🎉](http://w1.fi/cgit/hostap/commit/?id=209dad066e5275ac13f52623cc9eaf9b70910123)).
</p></aside>

On November 25, I told my contact at Ford that I have a [portable 802.11r lab](/assets/images/portable-80211r-lab.jpg) that I can bring to the dealership of her choosing.

On November 29, [I learned](/2016/11/29/email-1480450861.html) that the Regional Customer Service Manager that took this case on November 17 was changing positions and that my case would be handled by yet another representative, now my fourth contact at Ford corporate. (What kind of turnover do they have? Is this normal?)

On November 30, I brought my vehicle to a dealership per Ford's request, where they identify this as repair order number `6220998`. The dealership was not interested in my portable 802.11r lab, but the technician at the dealership's service center was able to confirm the issue using their internal wireless network, which apparently is also 802.11r-capable. They referred the case to some internal Ford hotline.

Denial
---

On December 5, [I received an email](/2016/12/05/email-1480967979.html) from the second Regional Customer Service Manager starting with:

> At this time it has been determined that the cause of the concern is the customer’s connection point and not a vehicle failure. There has been no diagnosis and therefore no repair remedy to apply in this case. As I am not a technical resource, I am unable to make any recommendations, but I must inform you that I have exhausted my technical resources and have no other recourse to follow.

[I replied](/2016/12/05/email-1480969276.html):

> This is not satisfactory.
> 
> I understand that it's convenient to blame the customer's equipment, but I have two different access points from two different vendors both showing that this vehicle is in violation of the IEEE 802.11 standard. Both access points can document their communications with my vehicle, and both show the vehicle sending an association request frame which the 802.11 standard says **the access point must reject**. Please explain how you conclude that this is a problem with my equipment and not a problem with the vehicle.

After [doubling down](/2016/12/05/email-1480969778.html), the Regional Customer Service Manager [informed me](/2016/12/05/email-1480970216.html) that the dealership had not requested that this case be reviewed by a Field Service Engineer, and that she could work with them to make that happen. [I insisted](/2016/12/05/email-1480972220.html) that the engineer be presented with the information contained in this document, saying in part:

> I want to know that some technical resource at Ford has actually **looked** at this information. If they did, I guarantee they would have more to say than "the cause of the concern is the customer’s connection point".

On December 20, [I wrote again](/2016/12/20/email-1482274095.html):

> It's been over two weeks since our last contact. On my end, I've made my writeup accessible via a shorter link: [http://ford-wifi-is-broken.com/](http://ford-wifi-is-broken.com/)
> 
> What's currently in progress at Ford? Has a Field Service Engineer been engaged on this case? When should I expect further updates?

I got an auto-response, which is what I expected given it's ~Christmas. I [got a reply](/2016/12/27/email-1482870832.html) on December 27:

> Details of your vehicle concerns were submitted by the dealership to Ford’s corporate technical resource for review. It has been determined that there is no vehicle failure, rather a concern with the connection point. Per technical assistance team, at this time we would have you reference the information on pages 105 and 106 of the second printing of the Sync 3 supplement. You may also wish to contact the Sync In-Vehicle Team for assistance on connecting to the Wi-Fi source at 800-392-3763, Option 3 when prompted.
> 
> Your case with us will be closed at this point as there has been no damage or defect found and our solution for you is to contact the Sync In-Vehicle Team.

…to which [I replied](/2016/12/27/email-1482873053.html):

> > Details of your vehicle concerns were submitted by the dealership to Ford’s corporate technical resource for review. It has been determined that there is no vehicle failure, rather a concern with the connection point.
> 
> This conclusion does not agree with the facts.
> 
> I've told everyone who will listen: this vehicle sends malformed association request frames when connecting to 802.11r-capable access points. I've sent diagrams showing these frames, along with the standards documents which explicitly say that those frames make no sense. I don't see any way to present this except that SYNC 3 has a bug, but don't take my word for it: the dealership service personnel told me they were able to reproduce this issue with the dealership's 802.11r-capable network too.
> 
> It's broken for me, and I've offered specifics as to precisely what is going wrong. I brought my vehicle to [dealership] as you requested, and they say it's broken for them too. After all this, it's insufficient for Ford corporate to close the case saying only it's "a concern with the connection point". Seriously – "connection point" isn't even a term used in the 802.11 standard, nor is that phrase used by anyone who's done any wireless troubleshooting.
>
I believe someone looked at this case, saw "can't connect to wifi", and closed it saying "must be the customer's problem". I wrote to [previous rep] about this possibility on November 17:
>
> > So, yes, the basic problem is that my vehicle can't connect to my wifi network. It's tempting to think that wifi problems can be dismissed as an incompatibility that's both or neither party's fault, but in this case that's simply not true. My vehicle can't connect to my wifi because it's trying to authenticate in a way that violates the 802.11 standard. Specifically, my vehicle can't connect because it's transmitting an authentication frame that is expressly prohibited by IEEE Std 802.11-2012 § 12.4.2:
> > 
> > …
> > 
> > Here's a picture of my vehicle sending an Association Request frame with an MDE and an RSN indicating AKM suite 00-0f-ac:2, which is exactly what that sentence says not to do:
> > 
> > …
> > 
> > **That** is the problem. Someone at Ford engineering should dig into the wifi section of the SYNC 3 software, where they'll find that the WPA supplicant does not support 802.11r, and that the component sending association requests is configured to request 802.11r anyway. **That** is why SYNC 3 would send this kind of invalid frame. They can either a) tell the component sending association requests not to request 802.11r, or b) keep requesting 802.11r and teach the WPA supplicant how to handle 802.11r. **That** is this fix.
> 
> Closing this case with that explanation is not satisfactory, and repeating that explanation without elaboration does nothing to challenge this interpretation of events.
>
> > You may also wish to contact the Sync In-Vehicle Team for assistance on connecting to the Wi-Fi source at 800-392-3763, Option 3 when prompted.
>
> I would love to contact the Sync In-Vehicle Team, but this phone number leads to a recording which says "Hello, thank you for calling It Cosmetics. If you are a new customer and would like to place your first order, please press 1." I pressed 3 and a woman answered who assured me that they only sell makeup. What is the correct number?
>
> > Your case with us will be closed at this point as there has been no damage or defect found and our solution for you is to contact the Sync In-Vehicle Team.
>
> You told me this on December 5. You also said that the dealership could refer this to a Field Service Engineer, and that you would recommend that the dealership do this. That's what I was following up about. Has that happened?

Google led me to 800-392-3673 (800-392-FORD), where I reached a representative who found this case and referred it to Tier 2 within the In-Vehicle Technology group. I hope that's a step in the right direction, though if is, I'll draw unpleasant conclusions about the Customer Care Team which has a) had this information for months and b) somehow never contacted the IVT group on my behalf.

I also got a call from the service desk at the dealership, a member of which was CC:ed on my reply, who informed me that the dealership is still trying to get the Field Service Engineer out to the dealership to address this issue. He's hopeful that the engineer will be more available now that the holiday season is winding down.

On December 29, I [dug into the SYNC 3 software](#software-analysis) and found additional information relevant to this issue. SYNC 3 v2.2's `wpa_supplicant` is never configured to attempt 802.11r authenticaiton, despite the vehicle attempting to associate to a 802.11r mobility domain.
