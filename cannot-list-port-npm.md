# The Problem

You get the following issue when running `npm start`, which effectively calls `ng serve --ssl true --host localhsot --port 3000`

```
Running into EACCES: permission denied exception when running "npm start" Windows 10
```

# Key Assumptions

- This problem only happens in Windows 10. The concept applies to other operating systems, but the exact solution commands work in Windows only.
- Some commands require Powershell Core 6.x

# Pre-Checks

Your problem might not exactly due to the main answer of this page, so let's make some quick checks:

1. Check if the port -- which is `3000` in our case -- used by any other process. Run the following command on a Powershell session: `Get-NetTcpConnection -LocalPort 3000`. If this command returns something, run this command to find out who is using that port `Get-Process -id $(get-nettcpconnection -localport 3000).OwningProcess`, and see if you can shut the process down.
1. Check if your Firewall has rules to explicitly ban that port. A command like this might help `netsh firewall show state`.

If nothing above applies to you, then go ahead and read the solution below.

# The Solution

## Check your excluded port range on Windows 10

Run the following command  to see the excluded port ranges:

```
netsh interface ipv4 show excludedportrange protocol=tcp
```

You will see something like below. Check if your port is in any of those ranges. If yes, then you identified the first part of the problem. Different applications for their current and future needs allocate ports. This unfortunately means that other application cannot use these ports. 

## Identify which application allocated the range you need

This part is hard and I don't have a solid way of finding out which applications are the owner of these ranges. You will have to shut down/uninstall applications that might look suspicious to you. In my case, it was `Hyper-V Service`. 

## Check it is Hyper-V service that allocates these ports

Simply run the following command to remove Hyper-V feature from your Windows. Don't worry, we are going to bring it back once we are done.

```
dism.exe /Online /Disable-Feature:Microsoft-Hyper-V
```

Now, restart your computer.

After your computer is rebooted, run this command to see if your port is removed from the excluded list:

```
netsh interface ipv4 show excludedportrange protocol=tcp
```

This is a good sign. Then, Hyper-V is the problem.

## Allocate the port range for yourself before Hyper-V does so

We will now allocate the port ranges we want for ourselves so that when Hyper-V is restarted, it won't be able to allocate them for itself. 

Run the following command and adjust its parameters to your needs:

```
netsh int ipv4 add excludedportrange protocol=tcp startport=2999 numberofports=10
```

Great. Now, you have the ports allocated for your needs. You can bind whatever application you want to these ports.


## Enable Hyper-V back again

Run the following command:

```
dism.exe /Online /Enable-Feature:Microsoft-Hyper-V /All
```