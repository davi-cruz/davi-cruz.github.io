---
layout: single
title: VyOS Lab Setup - Part 1 - Basic
category: vyos
tags: hyper-v vyos lab
---

Portugues

Hello everyone!

Often while supporting customers or learning a new technology is required to have a lab environment to test some scenarios.

Some years ago I met Vyatta project and since then I've been using it, as most recently the community fork [VyOS](https://vyos.io/) to run my lab environments in a easy way. 

Besides [VyOS Documentation](https://docs.vyos.io/en/latest/index.html) is very rich, I'm writing this series to share the setup I often use to test software, analyze malware and other tasks you might require a granular control over connectivity of your endpoints in a virtual lab.

For this series I'll show you how to do it using Hyper-V, once this is my main Operating System is Windows 10 and this feature is built-in and free to use on some Windows 10 SKUs.

# Hyper-V Setup
## Installing Hyper-V

For those interested on using Hyper-V on their Windows 10 boxes, you basically need to meet the following specs:

- A **supported Operating System Version** which should be Pro, Enterprise or Education vide [Microsoft Docs](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/reference/hyper-v-requirements);

- A **Hyper-V compatible Hardware**, which basically consists in processor support for virtualization, which can be checked by running `systeminfo` in a command prompt like the image below, extracted from Microsoft Docs:
  
  ![](C:\Repositories\DaviCruz\blog\assets\images\2020-12-19-vyos-lab-setup-part-1-basics\systeminfo-upd-1608406448646-1608406723692.png)



To get Hyper-V installed, you simply need to enable it through **Control Panel** > **Turn Windows Features on or off** selecting the components below:

- Hyper-V Platform
- Hyper-V Management Tools

It can also be done by running the following powershell cmdline in a elevated prompt:

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All
```

After installing these components and restarting the computer, you should be ready to start creating your virtual machines and networks.

## Configuring Networks

