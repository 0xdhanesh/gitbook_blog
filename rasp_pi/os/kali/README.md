---
aliases:
  - What not to do and how to do things right notes
tags:
  - RaspberryPi
  - Linux
  - Networking
---

# Kali Linux on Raspberry Pi

## WiFi - Why you are doing it wrong

When flashing Kali linux OS for Raspberry Pi from Pi Imager, most of us choose the standard installation process with customization option which provides us the ability to set the hostname, user management and wifi connection capabilities.

This particular approach will end up creating a conflict with Kali Linux's NetworkManager, and we end up with a os without Wifi Capabilities, but other radio communications like Bluetooth and the physical network connection (Ethernet) will work fine

### Solutions

* Easy way: Just **Skip Customization** and continue with flashing the SD Card / SSD. The default creds **kali:kali** will be applied with the hostname **kali-raspberrypi**

![](../../../.gitbook/assets/Pi_Imager_Setup.png)

* Hard way: Manually overwriting the settings to disable the one set by the Pi Imager and kick starting the service

## Hardway - Manual Over-Ride (╥﹏╥)

Here while solving this we will take a look at Netplan, cloud-init, NetworkManager, wpa\_supplicant, interface ownership

The failure of initializing the wifi was **not** because of a blocked RF device / missing drivers. This can be confirmed with `rfkill` command

{% hint style="info" %}
From Manage:&#x20;

```bash
rfkill - tool for enabling and disabling wireless devices
```
{% endhint %}

The output from rfkill command will let us know if the device is either hard / soft blocked. The NetworkManger's log will tell us exactly what the issue is, when I faced this issue I tried to look at the logs with the following command:

```bash
sudo journalctl -b -u NetworkManager --no-pager | grep -iE 'wlan0|wifi|supplicant|wireless|firmware|brcm'
```

From the output of journalctl we got the message "_Couldn't initialize supplicant interface, supplicant interface keeps failing._" Then by looking at the running processes of _wpa\_supplicant_, for the first time I got two instances of wpa\_supplicant running, and for the second time it was none. Either multiple instances or no instance at all. This behaviour can be confirmed with the following commands:

```bash
ps -ef | grep '[w]pa_supplicant' # either 2 instance or 0 instances as output
sudo systemctl --type=service --all | grep -i wpa 
```

This confirms that the actual failure was control-plane contention: two user-space networking stacks attempting to own the same wireless interface or none.

Remember when we provide the information about the WiFi on the Pi Imager, it asks for the WiFi SSID and the password, those details gets stored on the "_/etc/netplan/\*.yaml_" (typically, the name of the file will be 50-cloud-init.yaml). This file contains all WiFi definition. Netplan renders that definition via **networkd** path and launches an interface specific `wpa_supplicant` process using the _/run/netplan/wpa-wlan0.conf_, at the same time the desktop expected NetworkManger to manage `wlan0`. NetworkManager could see the device, but cannot claim the interface as its own supplication integration hinders it. Thus we get the WiFi not available message from the status bar.

> \[!NOTE] Inspection Btw, the network devices can be inspected with the following command
>
> ```bash
> nmcli device / nmcli device show 
> ```
>
> We will use `nmcli` a lot

During this conflict, the output of nmcli will show either `unavailable` or `unmanaged` for `wlan0`

### Solution Steps

The issue can be resolved by restoring the single ownership of the interface, which can be achieved by the following steps:

* Stop cloud-init from regenerating network configuration
* Make Netplan delegate networking to `NetworkManager`
* Remove the stale per-interface Netplan WiFi Definition
* Allow NetworkManager's single D-Bus-managed `wpa_supplicant` instance to own `wlan0`

> \[!NOTE] nmcli patch: If you search the internet / ask AI for solution do not rely only on the `sudo nmcli device set wlan0 managed yes` as this is only a temporary patch and when the Pi reboots we will be presented with the same issue and the command may not work for the second time
