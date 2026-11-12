# Android-Wireless-Security-Assessment-Lab
Hands-on project · Android 15 · Kali Linux · Network Security
Android Wireless Security Assessment Lab

Kali Linux | Hyper-V | Android 15 | ADB | Nmap | scrcpy

Project Overview

For this project, I wanted to see if I could connect to my Android tablet remotely using Kali Linux and learn more about how Android handles network connections, remote access, and camera permissions.

I used my Lenovo Tab M11 running Android 15 and Kali Linux inside a Hyper-V virtual machine on my Windows PC.

My main goal was to see what ports were open on the tablet, figure out how to connect to it over Wi-Fi, and eventually control and view the tablet's screen directly from Kali without using a USB cable.

I also wanted to understand how Android camera permissions work and how to tell if an app is actually using the camera.

I ran into a few problems along the way, mainly with the network configuration, ADB pairing, and Linux permissions, but I was able to troubleshoot them and get everything working.

All testing was done on my own tablet and home network.

My Lab Setup

Component

What I Used

Virtual machine

Microsoft Hyper-V

Operating system

Kali Linux

Android device

Lenovo Tab M11 (TB330FU)

Android version

Android 15

Network

Home Wi-Fi and Hyper-V virtual network

Network scanning

Nmap 7.99

Android debugging

ADB 1.0.41

Screen mirroring

scrcpy 4.1

What I Wanted to Accomplish

Find my Kali Linux IP address and understand how its network was configured.

See if Kali could communicate with my Android tablet.

Scan the tablet to see if any TCP ports were open.

Connect Kali to the tablet using ADB over Wi-Fi.

Check which apps had camera permissions.

See if I could identify when the tablet's camera was being used.

Remotely view and control my tablet from Kali.

Phase 1 — Figuring Out My Network

I started by checking the network configuration on Kali Linux.

I ran:

ip -br addr
ip route

This showed me my active network interface, IP address, and default gateway.

I noticed Kali was using an IP address in the 172.21.x.x range, while my tablet was connected to my home Wi-Fi using an address in the 192.168.1.x range.

At first, I wasn't sure if they could communicate since they were on different subnets.

To check how Kali would reach the tablet, I ran:

ip route get <TABLET_IP>

The output showed that Kali would send traffic through its Hyper-V gateway.

Testing the Connection

Next, I tested whether Kali could actually reach the tablet.

ping -c 4 <TABLET_IP>

My results were:

4 packets sent

4 packets received

0% packet loss

Average latency around 93.8 ms

This confirmed that Kali could communicate with the tablet even though they were on different networks.

What I learned: Two devices don't always have to be on the same subnet to communicate. As long as there's proper routing between them, they can still exchange traffic.

Phase 2 — Scanning My Tablet With Nmap

Now that I knew the tablet was reachable, I wanted to see what services it was exposing over the network.

I used Nmap:

nmap -sT -sV -Pn <TABLET_IP>

The scan checked 1,000 common TCP ports.

My Results

The tablet was online and reachable.

Nmap scanned 1,000 TCP ports.

All 1,000 ports were closed.

No open TCP services were detected during that scan.

At first, I thought this might mean I couldn't connect to the tablet at all.

But I learned that just because those ports are closed doesn't mean the device can't communicate over the network. It just means there wasn't a service listening on those specific ports when I scanned them.

I also learned that Nmap's default scan doesn't check every possible port, so this wasn't a complete security assessment.

Phase 3 — Connecting to Android Using ADB Over Wi-Fi

This was probably the part where I ran into the most problems.

I wanted to connect directly to my tablet from Kali without plugging in a USB cable.

I found out Android has a feature called Wireless Debugging, which allows you to connect using Android Debug Bridge (ADB).

I enabled Developer Options and Wireless Debugging on my tablet.

First, I checked whether ADB was installed:

adb version

ADB was already installed, so I moved on to pairing.

My First Problem — ADB Pairing Errors

When I first tried pairing, I kept getting an error:

error: protocol fault (couldn't read status message): Success

I eventually figured out I was confusing the pairing code with the port number.

I also wasn't always including the correct port in my command.

The correct command format was:

adb pair <TABLET_IP>:<PAIRING_PORT>

Android displays a pairing port and a separate six-digit pairing code.

Before trying again, I checked whether Kali could reach the pairing port:

nc -vz -w 5 <TABLET_IP> <PAIRING_PORT>

The result showed the port was open.

I tried pairing again using the correct port and code.

This time it worked.

My Second Problem — Pairing Was Successful but ADB Wouldn't Connect

Even after successfully pairing, I couldn't establish the actual ADB connection.

I kept getting:

failed to connect

I learned that Android uses a separate connection port after pairing, and that the ports can change when Wireless Debugging restarts.

I checked the IP address and connection port shown in Android's Wireless Debugging settings and tested the TCP connection again.

Once I had the correct port, I ran:

adb connect <TABLET_IP>:<CONNECTION_PORT>

This time Kali successfully connected.

To verify it, I ran:

adb devices -l

The tablet appeared with the status:

device

That confirmed I had an authorized wireless ADB connection.

What I learned: ADB pairing and ADB connecting are two different steps. Pairing authorizes the computer, while connecting establishes the actual debugging session.

I also learned that basic network connectivity, TCP connectivity, and application authentication are different things.

Phase 4 — Checking Android Camera Permissions

After connecting through ADB, I wanted to see how Android handles camera permissions.

I started by examining which apps requested camera access and whether Android had granted those permissions.

One of the apps I checked was Zoom.

I ran:

adb shell dumpsys package us.zoom.videomeetings

I found this in the output:

android.permission.CAMERA: granted=true

This showed that Zoom had permission to access the camera.

But I wanted to know whether having permission meant the app was actually using it.

Checking Camera Activity

I ran:

adb shell dumpsys media.camera

The results showed:

Number of camera devices: 2
Active Camera Clients:
[]

Both cameras were closed at the time I checked.

The logs also showed that a Lenovo application had connected to and disconnected from one of the cameras earlier.

What I Learned

This helped me understand that there is a difference between:

An application requesting camera permission.

Android granting that permission.

An application actually opening and using the camera.

Just because an app has camera permission doesn't mean it's currently recording anything.

I also learned that Android keeps diagnostic information about camera activity that can be inspected through ADB.

I didn't find or demonstrate a way to bypass camera permissions. This part of the project was about understanding how Android manages camera access.

Phase 5 — Viewing and Controlling My Tablet Remotely

Once ADB was working, I wanted to take it a step further.

My goal was to see my tablet's screen live on Kali and control it using my keyboard and mouse.

I used a program called scrcpy, which allows Android screen mirroring through ADB.

My First Problem — scrcpy Wouldn't Open

I initially ran:

scrcpy -s <TABLET_IP>:<CONNECTION_PORT>

But I got errors saying:

XDG_RUNTIME_DIR is invalid or not set
Authorization required, but no authorization protocol specified
Could not initialize SDL video

At first, I thought it might be another network or ADB issue.

But I eventually realized I was running Kali as the root user, and root wasn't properly connected to my graphical desktop session.

How I Fixed It

I exited root:

exit

Then I checked my graphical environment:

echo $DISPLAY
echo $XDG_RUNTIME_DIR

The results were:

:1.0
/run/user/1000

This showed that my regular Kali user had the graphical environment configured.

I ran scrcpy again as the normal Kali user:

scrcpy -s <TABLET_IP>:<CONNECTION_PORT>

And it worked.

I could now see my Android tablet's screen live on Kali Linux and control it remotely over Wi-Fi.

I didn't need a USB cable, and the connection was working through my existing ADB setup.

What I learned: Not every error is a networking issue. Sometimes the problem is related to Linux user permissions or the desktop environment.

This was a good example of why it's important to understand what an error message actually means instead of just trying random commands.

Skills I Practiced

Networking

Understanding IPv4 addresses and subnets

Identifying network interfaces

Checking routing tables and default gateways

Testing connectivity with ping

Testing TCP ports with Netcat

Scanning devices using Nmap

Troubleshooting connections between different networks

Cybersecurity

Performing network reconnaissance on my own device

Identifying exposed TCP services

Understanding Android runtime permissions

Checking camera-service activity

Learning how authorized remote access works

Understanding the difference between open ports, permissions, and active services

Linux and System Administration

Working with Kali Linux commands

Managing Android devices using ADB

Understanding root versus regular-user permissions

Troubleshooting Linux graphical display issues

Running remote administration tools

Working with virtual machine networking

Biggest Things I Learned

One of the biggest things I learned from this project is that having network connectivity doesn't automatically mean you can access a service on another device.

For example, I was able to ping my tablet, but Nmap showed that the 1,000 ports it scanned were closed.

I also learned that opening a TCP connection doesn't mean you're authenticated or authorized to use the service.

Another important thing was understanding Android's permission system. An application having camera permission doesn't mean the camera is currently active.

I also got more comfortable reading terminal errors and figuring out what was actually causing them.

The scrcpy problem was a good example because I originally thought the connection was failing, but the real issue was that I was running it as root without access to the graphical session.

Security Considerations

I performed all testing on my own Android tablet using my home network.

Wireless Debugging required me to enable the feature and authorize Kali through Android's pairing process.

I didn't bypass Android's camera permissions or exploit any vulnerabilities.

Since ADB provides powerful access to Android devices, I learned why it's important to disable Wireless Debugging when it's no longer needed.

Final Results

By the end of this project, I was able to:

Connect Kali Linux to my Android tablet across different subnets.

Scan the tablet and inspect its TCP port exposure.

Successfully pair and connect using wireless ADB.

Check Android camera permissions and camera-service activity.

Troubleshoot ADB connection failures.

Fix Linux graphical display permission errors.

Remotely view and control my Lenovo tablet from Kali using scrcpy.

My original goal was to learn how to connect to and interact with an Android device remotely.

By the end, I had a better understanding of networking, Android security, Linux troubleshooting, and how different tools work together.
