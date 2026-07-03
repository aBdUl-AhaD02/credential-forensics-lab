Here's a description you can drop in:
This lab walks through a real-world offboarding failure and what happens when nobody catches it. Two employees got fired, but their accounts stayed active — nobody flagged it, nobody killed the sessions. The company suspected they'd used that leftover access to grab private files off the server before anyone noticed, and the job was to prove it actually happened.
All I had to work with was a small network capture and a memory dump of LSASS. That's genuinely enough if you know where to look.
Started by ripping through the pcap in Wireshark, filtered do
