# check_vmware_snapshots
nagios check using VMware PowerCLI for snapshot health

# Requirements
PowerShell for Linux and VMware PowerCLI installed on nagios server

# Installation Steps

Install PowerShell on the Linux-based nagios server.

More details at https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell-on-linux
```
which yum && yum install https://github.com/PowerShell/PowerShell/releases/download/v7.4.6/powershell-7.4.6-1.rh.x86_64.rpm
which apt && apt install https://github.com/PowerShell/PowerShell/releases/download/v7.4.7/powershell_7.4.7-1.deb_amd64.deb
```

Add VMware PowerCLI from the PowerShell Gallery
```
[root@linuxbox]# pwsh
PowerShell 7.4.6
PS /home/someuser> Install-Module -Name VMware.PowerCLI -Scope AllUsers
```

Disable the PowerCLI call-home to VMware for usage statistics reporting.
```
[root@linuxbox]# pwsh
PowerShell 7.4.6
PS> Set-PowerCLIConfiguration -Scope AllUsers -ParticipateInCEIP $false
```

## create vCenter userid

Create a read-only user account on the vCenter server.  

The nagios check will use these credentials to query vCenter for all the snapshots.

Login to vCenter web interface, click Administration menu, Users and Groups

<img src=images/adminmenu.png>




Set the domain to vsphere.local, click Add.

<img src=images/usersandgroups.png>




Enter the user details, click Add.

<img src=images/userdetails.png>

## assign read-only role to vCenter userid
Now we will assign a role to the nagios user so it has sufficient privilege to view snapshots.

Note that this only gives the userid on the vCenter server enough permission to view snapshots, but not to create or delete snapshots.

Click Global Permissions, Add.

Set the domain to vsphere.local

Set the user to nagios

Set the Role to Read-Only

Select the checkbox to Propagate to children  (to push the permission out to the ESXi hosts)

Click OK

<img src=images/changerole.png>


# Nagios configuration

Copy the check script and config file to the appropriate plugins folder and confirm the script is executable
```
su - nagios
cd /tmp
git clone https://github.com/nickjeffrey/check_vmware_snapshots
cd check_vmware_snapshots
cp check_vmware_snapshots.cfg /usr/local/nagios/libexec/check_vmware_snapshots.cfg
cp check_vmware_snapshots     /usr/local/nagios/libexec/check_vmware_snapshots
chown nagios:nagios           /usr/local/nagios/libexec/check_vmware_snapshots.cfg
chown nagios:nagios           /usr/local/nagios/libexec/check_vmware_snapshots
chmod 744                     /usr/local/nagios/libexec/check_vmware_snapshots.cfg
chmod 755                     /usr/local/nagios/libexec/check_vmware_snapshots
```


Edit the default config file at /usr/local/nagios/libexec/check_vmware_snapshots.cfg, adjusting as appropriate for your environment.
```
# config file for check_vmware_snapshots script
# lines beginning with a hash mark # are ignored
# adjust the variables below to match your environment
vcenter=vcsa.example.com
username=nagios@vsphere.local
password=MySecretPassw0rd!
max_gb_per_snapshot=10
max_snapshots_per_vm=3
max_snapshot_age_days=7
```

Create a section similar to the following in the commands.cfg file on the nagios server.
```
# 'check_vmware_snapshots' command definition
define command {
      command_name    check_vmware_snapshots
      command_line    $USER1$/check_vmware_snapshots 
      }

```

Create a section similar to the following in the services.cfg file on the nagios server.  This will use the default config file.
```
define service {
       use                             generic-service
       host_name                       vcenter.example.com
       service_description             VMware snapshots
       check_interval                  120              ; Actively check the host every 120 minutes
       check_command                   check_vmware_snapshots
       }
```


HINT: if you have multiple vCenter hosts, you can specify a custom config file for each stanza as shown below.
```
define service {
       use                             generic-service
       host_name                       vcenter1.example.com
       service_description             VMware snapshots
       check_interval                  120              ; Actively check the host every 120 minutes
       check_command                   check_vmware_snapshots!/usr/local/nagios/libexec/check_vmware_snapshots.vcenter1.cfg
       }

define service {
       use                             generic-service
       host_name                       vcenter2.example.com
       service_description             VMware snapshots
       check_interval                  120              ; Actively check the host every 120 minutes
       check_command                   check_vmware_snapshots!/usr/local/nagios/libexec/check_vmware_snapshots.vcenter2.cfg
       }

```



# Sample output

If everything is ok, output will look similar to:
```
VMware snapshots OK total_snapshots:7  largest_single_snapshot:12 GB , total_size:23 GB , oldest_snapshot:4 days
```

If there are problems, each problem will be displayed in the nagios check output.
Please note that if there are many problems with old snapshots, the output message may be very long.  
To make the following example easier to read, each problem has been printed on a separate line.
```
VMware snapshots WARN, 5 issues found, please cleanup snapshots.   
'dbserv' snapshot     'VM Snapshot 1/23/2025, 8:24:12 PM' is 12 GB in size, max is 10 GB. 
'dbserv' snapshot     'VM Snapshot 1/23/2025, 7:16:25 PM' is 4 days old, which exceeds max age of 3 days. 
'dbserv' snapshot     'VM Snapshot 1/23/2025, 8:24:12 PM' is 4 days old, which exceeds max age of 3 days. 
'fileserver' snapshot 'VM Snapshot 1/23/2025, 7:17:35 PM' is 4 days old, which exceeds max age of 3 days. 
'dbserv' has 5 snapshots, max is 3.
 total_snapshots:4  largest_single_snapshot:12 GB , total_size:23 GB , oldest_snapshot:4 days
```

`
