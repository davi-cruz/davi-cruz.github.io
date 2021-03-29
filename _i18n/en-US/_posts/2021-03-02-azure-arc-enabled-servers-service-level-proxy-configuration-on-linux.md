---
layout: single
title: "Azure Arc enabled Servers: Service-level proxy configuration on Linux"
namespace: "azure-arc-enabled-servers-service-level-proxy-configuration-on-linux"
category: "Azure Arc"
tags: azure arc linux
date: 2021-03-02 18:00:00
---

Sometimes, when engaged to customers, I use to be asked to help them to configure proxy for internet services in some specific services, and this is not different with Azure Arc

In this post I'll guide you on how to configure this for Linux, without configuring system wide.

## Default configuration

By default, when we specify a proxy server during the Azure Arc agent installation in a Linux Server, there's a utility which handles all these requestes, which is `azcmagent_proxy, as below:

```bash
dcruz@vmlx02:~$ sudo azcmagent_proxy
Usage:  azcmagent_proxy add <URL> - to add URL as the proxy
        azcmagent_proxy remove - to delete configured proxy
dcruz@vmlx02:~$
```

When we define the proxy using it, two files are changed, as we can see in the example below:

```bash
dcruz@vmlx02:~$ sudo azcmagent_proxy add http://vmlx01:3128
    No proxy previously configured
Removing proxy environment variable from file:  /opt/azcmagent/bin/azcmagent
    No proxy previously configured
Setting proxy environment variable to file:  /lib/systemd/system.conf.d/proxy.conf
Adding proxy environment variable to file:  /opt/azcmagent/bin/azcmagent
dcruz@vmlx02:~$
```

The first file, `/lib/systemd/system.conf.d/proxy.conf`, defines the proxy to all systemd daemons globally, while the `/opt/azcmagent/bin/azcmagent` is the wrapper for the Azure Arc command line utility.

## Changing proxy settings only for Azure Arc

In order to configure the services to have connectivity needed to perform all management tasks, without enabling it system wide, is necessary to change each of the **systemd** unity files, as described below;

- If you have already configured proxy using `azcmagent_proxy` during the setup and desires to rollback its configurations, is necessary to run it again to do the task:

  ```bash
  sudo azcmagent_proxy remove
  ```

- Also, for each service unity used by Azure Arc for Linux Servers (`himdsd.service` ,`gcad.service` ,`extd.service`) located at `/lib/systemd/system`, we need to add a variable in the section `[Service]` defining **https_proxy**, as the example below and can also be seen in the official [**systemd** Documentation](https://www.freedesktop.org/software/systemd/man/systemd.service.html):

  ```ini
  [Service]
  # [...]
  Environment=https_proxy=http://vmlx01:3128
  ```

- After changing all the three mentioned files, run the commands below to reload daemons and restart the services

  ```bash
  sudo systemctl daemon-reexec
  sudo systemctl restart extd.service himdsd.service gcad.service
  ```
  
- Also is necessary to add the proxy to the `azcmagent` wrapper, in order to make the command `azcmagent connect` work properly in the system. This is achieved y adding a line `export https_proxy=<proxyserver>` right below the commented message that a specific line should not be removed in the file located at `/opt/azcmagent/bin/azcmagent`, as the sample below:

  ```bash
  [...]
  
  # Do not remove this line ==== place Environment Variables below ======
  export https_proxy=http://vmlx01:3128
  
  [...]
  ```

After making these changes, you should see the following line on your logs, which is proof that the service is using the correct proxy configuration inside the  `/var/opt/azcmagent/log/himds.log`

```log
time="yyyy-MM-dd02T17:34:07Z" level=debug msg="Using Https Proxy: http://vmlx01:3128"
```

At this point you're ready to manage this endpoint using Azure Arc or connect it using the command `azcmagent connect` in case it is a new install.

### Script

To simplify configuration, I've created one script inspired on `azcmagent_proxy` to automate this configuration for Linux distributions using **systemd**, available at [Security/azcmagent_proxydaemon.sh at main · davi-cruz/Security (github.com)](https://github.com/davi-cruz/Security/blob/main/AzureArc/azcmagent_proxydaemon.sh).

This could be useful to you not only to define proxy settings for Azure Arc enabled Linux Servers but also for any service unities you run in your workloads :smile:

:warning: **Alert**:
As these changes are not global, other services installed by using Azure Extensions (Native or Custom ones) might require additional changes in order to make then work with the default proxy or to also make the same configuration to their services.
{: .notice--warning}

HTH!
