# Overview

ProxyCommand is used to allow SSH to private linux instance without storing ssh key on bastion

https://en.wikibooks.org/wiki/OpenSSH/Cookbook/Proxies_and_Jump_Hosts#ProxyCommand_with_Netcat


# Step By Step

1. Setup VPC using Wizard with 2 subnets: Public and Private

2. Amazon linux Bastion in public subnet:

  * uses bastion-key.pem
  * SecurityGroup: allow SSH from your local IP

3. Amazon linux Private instance in private subnet: 
  
  * uses private-key.pem
  * SecurityGroup: allow SSH from Bastion private IP

4. Edit ssh_config file (~/.ssh/config) with following setting:
```
Host bastion <- host shortcut name
   HostName <<Bastion_Public_IP>>
   User ec2-user 
   IdentityFile <<local_path>>/bastion-key.pem  <- or whatever key you use for bastion
   ProxyCommand none
Host private-instance
   HostName <<Private_Instance_Private_IP>>
   User ec2-user 
   IdentityFile <<local_path>>/private-key.pem <- or whatever key you use for private instance
   ProxyCommand ssh bastion -W %h:%p 
```
Use Windows Putty -> go here https://stackoverflow.com/questions/28926612/putty-configuration-equivalent-to-openssh-proxycommand

5. Open terminal

```
ssh private-instance
```

6. BOOM!

