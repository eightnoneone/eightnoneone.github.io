---
layout: post
title:  "DNS over VPN with macOS WiFi"
date: 2026-03-08
tags: macos dns networking asa
---

I was having trouble with my MacBook Pro regularly disconnecting from my household WiFi and interrupting 
network Time Machine backups to my Synology NAS. Beyond the black box that is Time Machine troubleshooting, I 
could consistently see it complain that the network connection was dropped. Something odd was going on.

My first step was to see if could observe this happening from the network. 
The network is all Ubiquiti Unifi and with their Unifi Network 10 console, it's pretty easy to see client connection behaviors.

My MacBook was clearly doing a disconnect and reconnect every 30 minutes or so. I noticed that the three other MacBooks 
in the house never did a `disconnect`. They would occasionally _roam_ from one access point to another, 
but they didn't do a disconnect/reconnect to the same AP.

One unique thing about my computer is that it's often connected to a Cisco AnyConnect VPN for work. It's a split-tunnel,
so only specific IP ranges are sent over the tunnel. I've had Time Machine problems with this before as the DNS 
config would fail for my local mDNS names like `synology.local`. I had worked around this by configuring the 
Time Machine to use a static IP address, thus no DNS resolution needed.

While watching the macOS Console log for `airportd`, I found a flurry of activity that coincided with the Unifi console showing a reconnect.

```
14:52:04.622993-0500	airportd	[corewifi] LQM-WiFi: Stats12(5G): bt5g_status=0x3 bt5g_defer_cnt=0 bt5g_no_defer_cnt=0 bt5g_defer_max_switch_dur=0 bt5g_no_defer_max_switch_dur=0 bt5g_switch_succ_cnt=0 bt5g_switch_fail_cnt=0 bt5g_switch_reason_bm=0	default
14:52:07.152357-0500	airportd	DNSFailureSymptom detected on 'en0', notifying WiFiUsageMonitor w/fault:'kWiFiUsageFaultReasonSlowWiFiDnsFailure'	default
14:52:07.152393-0500	airportd	<airport[412]> DNSFailureSymptom detected on 'en0', notifying WiFiUsageMonitor w/fault:'kWiFiUsageFaultReasonSlowWiFiDnsFailure'	default
14:52:07.152564-0500	airportd	-[WiFiUsageMonitor canStartLQMAnalysisforTrigger:andReason:onWindow:] - Cannot start WiFiUsageLQMWindowAnalysis for (null) (max number of concurrent analysis (1) reached: 1)	default
14:52:07.153378-0500	airportd	-[WiFiUsageMonitor canStartLQMAnalysisforTrigger:andReason:onWindow:] - Cannot start WiFiUsageLQMWindowAnalysis for (null) (max number of concurrent analysis (1) reached: 1)	default
14:52:09.705075-0500	airportd	-[WiFiUsageMonitor setLinkEvent:isInvoluntary:linkChangeReason:linkChangeSubreason:withNetworkDetails:forInterface:]_block_invoke - isUp:NO details:<private>	default
14:52:09.713140-0500	airportd	WiFiUsageBssSession:: ChannelAfterRoam=0; ChannelAtJoin=40; FaultReasonApsdTimedOut=0; FaultReasonArpFailureCount=0; FaultReasonBrokenBackhaulLinkFailed=0; FaultReasonDhcpFailure=0;	default
14:52:09.716439-0500	airportd	WiFiUsageBssSession:: FaultReasonDnsFailureCount=0; FaultReasonL2DatapathStallCount=34; FaultReasonLinkTestDNSCheckFailure=0; FaultReasonLinkTestInternetCheckFailure=0; FaultReasonLinkTestLocalCheckFailure=0;	default
14:52:09.717561-0500	airportd	WiFiUsageBssSession:: FaultReasonRTTFailureCount=0; FaultReasonShortFlowCount=0; FaultReasonSiriTimedOut=0; FaultReasonSlowWiFi=0; FaultReasonSlowWiFiDUT=0; FaultReasonSymptomDataStallCount=0;	default
14:52:09.718892-0500	airportd	WiFiUsageBssSession:: FaultReasonUserOverridesCellularOutranking=0; FullScanCount=0; NetworkBssApMode=2; NetworkBssBand=5; NetworkBssChannel=40; NetworkBssChannelWidth=40; NetworkBssHasAppleIE=0;	default
14:52:09.741538-0500	airportd	WiFiUsageBssSession:: RoamConfigTriggerRssi=-75; RoamInMotion=0; RoamLatencyMax=0; RoamLatencyMin=0; RoamNeighborsCountTotal=0; RoamPingPongNth=0; RoamReasonBeaconLostCount=0; RoamReasonBetterCandidateCount=0;	default
14:52:09.747457-0500	airportd	WiFiUsageBssSession:: RoamReasonBetterConditionCount=0; RoamReasonDeauthDisassocCount=0; RoamReasonHostTriggeredCount=0; RoamReasonInitialAssociationCount=0; RoamReasonLowRssiCount=0;	default
14:52:09.755233-0500	airportd	WiFiUsageBssSession:: RoamReasonMiscCount=0; RoamReasonReassocRequestedCount=0; RoamReasonSteeredByApCount=0; RoamReasonSteeredByBtmCount=0; RoamReasonSteeredByCsaCount=0; RoamScanCountHighRssi=0;	default
14:52:09.776092-0500	airportd	-[WiFiUsageLinkSession linkStateDidChange:isInvoluntary:linkChangeReason:linkChangeSubreason:withNetworkDetails:]: link session ended for NetworkName:REDACTED BSSDetails:{BSSID:<redacted>,Channel:40(5 Ghz),RSSI:-53}, deferred count 0	default
14:52:09.776628-0500	airportd	-[WiFiUsageNetworkSession linkStateDidChange:isInvoluntary:linkChangeReason:linkChangeSubreason:withNetworkDetails:]: network session started	default
14:52:09.778079-0500	airportd	-[WiFiUsagePoorLinkSession startTimerWithTimeout:reason:]: start timer (LinkDownNotTDTimer) wait up to 90 secs	default
```

The fun part is `DNSFailureSymptom ... w/fault:'kWiFiUsageFaultReasonSlowWiFiDnsFailure'`. It appears that the macOS network driver is doing DNS
queries as a measure of the WiFi connection health. The VPN connection is a split-tunnel, but it is configured to tunnel _all_ DNS queries.
MacOS is testing DNS responses expecting local LAN performance when they are going over the VPN. The `FaultReasonL2DatapathStallCount=34` increases
throughout the connection period and never gets much over 30-35. I wasn't able to find any documentation on the exact definition of that counter, but
it seems to align with the reset.

Lucky for me, I'm the admin of the Cisco VPN, so I could make the change to only tunnel specific DNS queries:

`split-tunnel-all-dns disable`

WiFi bliss. No more connection resets!
