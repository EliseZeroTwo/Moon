---
layout: post
title: "TP-Link HS100 Denial Of Service"
date: 2019-01-26
excerpt: "Performing denial of service attacks on a smartplug, by abusing TP-Link's bad encryption"
tags: [vuln]
comments: false
---
### Introduction
The TP-Link HS100 is a smart plug in TP-Link's Kasa smarthome lineup

To control the plug you use the Kasa app. To setup the plug, it creates its own ad-hoc network that you connect to and then you configure it to connect to your home network.

The Kasa app works while connected to the internet, or to your local network as long as you are authenticated to your Kasa account (which is a firebase container).

### Research
I decided to capture some network traffic off my phone using `tPacketCapture Pro`. When inspecting the pcap I noticed a few packets being sent to the plug on port 9999, which turns out is the port the plug uses for communication.
![](https://i.imgur.com/8ZvOWtZ.png)

However the data was encrypted. Decompiling the Kasa apk and looking around we find an encryption algorithm. Remade in python it looks like the following
```python
    key = 171 #starting key will always be 171
    payload = pack(">I", len(cmd))
    for x in cmd:
        encb = key ^ ord(x)
        key = encb
        payload += chr(encb)
```
It just xors the key with every character in the command then the next key is the result of that.
Cool, now we have a way to encrypt commands to send to the device. Using this we can also decrypt the commands by rearranging the code to get the following
```python
    key = 171 #starting key will always be 171
    res = ""
    for x in result:
        decb = key ^ ord(x)
        key = ord(x)
        res += chr(decb)
```

We can now send the command to turn off the power output in the device from the Kasa App and look at what it is. The device uses JSON for communication and the decrypted command for turning the device off is the following: `{"system":{"set_relay_state":{"state":0}}}` and to turn the device on is `{"system":{"set_relay_state":{"state":1}}}`.

I mean all of this is pretty much useless because we can just send the encrypted data of the packet to the device avoiding all of this researching into how the commands work, but it was fun.

### Attack
Using what we found in the last section we can now craft a script to remotely perform denial of service on the device, either forcing it to never turn the power output on or off. Either works but personally I prefer forcing it to turn off. [Here is a python script that does this](https://gist.github.com/EliseZeroTwo/cb72c374c58136ff9d7a0f1a07087a52). No matter what, the device cannot be operated while the script is being ran. This works from the local network OR FROM AN EXTERNAL NETWORK (if 9999 is open)!!

Another cool thing we can do is replace the command with `{"system":{"reset":{"delay":1}}}` to do a factory reset on the device and this can be done from a local network or from an external network if the port is open.

### Futher Reading
The following is just research I did after finding all this out in trying to get code execution.

Unlike the HS100 counterpart (the HS110) the firmware files for the HS100 are not available directly on the project page. Luckily we know that there is too much trust going on between the Kasa app and the device. The Kasa app sends a command to the device which tells the device to download an update from the url provided as an argument in the command. Using this knowledge we log the network traffic and ask Kasa to update our device then unplugging the device from the wall as soon as it starts downloading it. This is safe because it does not flash anything until the firmware is fully downloaded. When decrypting the packet that tells the device to update the firmware we see this:

```
{"system":{"download_firmware":{"url":"http://download.tplinkcloud.com/firmware/hs100v1_eu_1.2.5_Build_171213_Rel.101415_2017_1516879250467.bin"}}}
```

So we now have a copy of the firmware. Putting this through binwalk we find out that it is U-Boot, a Linux Kernel, and a Sqaushfs. The squashfs does not have much in it, just BusyBox and a few other things (I have not spent more than 5 minutes looking around in it due to being busy). Cracking the login info using John The Ripper we find out that the login info is `root:media`. 

This part is still work in progress, please check back for updates.

