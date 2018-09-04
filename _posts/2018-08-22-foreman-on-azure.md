---
layout: post
title: The Foreman on Azure
---

This post is just a collection of notes regarding how I got The Foreman working in Azure on CentOS 7.5.

# Install the Puppet Master

1. It turns out The Foreman requires a forward DNS lookup of it's hostname, to get that working in Azure without DNS servers, I used a private zone for Azure DNS as per: https://docs.microsoft.com/en-us/azure/dns/private-dns-getstarted-powershell.

2. I then had to change my hostname as per: https://www.tecmint.com/set-change-hostname-in-centos-7/.

3. I then install The Foreman as per: https://theforeman.org/manuals/1.18/index.html#2.Quickstart.

4. I also had to open port 443 for inbound traffic to my VM over the public IP. This is only if you want to use The Foreman website remotely.

5. (optional) Install PDK (Puppet Development Kit): sudo yum install pdk

# Install a Puppet Agent

1. Change the hostname per: https://www.tecmint.com/set-change-hostname-in-centos-7/.

2. Install the Puppet Agent: sudo yum install puppet-agent

3. Start the service: /opt/puppetlabs/bin/puppet resource service puppet ensure=running enable=true

4. Approve the certificate on the Puppet Master

5. Test the agent: sudo /opt/puppetlabs/bin/puppet agent --test

# Jumpbox

If you need to SSH through a jumpbox (example shows connecting to a machine on the local network as 10.0.0.4):

ssh jumpbox.eastus2.cloudapp.azure.com -o StrictHostKeyChecking=no -o ProxyCommand='ssh jumpbox.eastus2.cloudapp.azure.com -o ChallengeResponseAuthentication=yes -p 22 -W 10.0.0.4:22'

# Helpful References

* https://www.example42.com/tutorials/PuppetTutorial
* https://github.com/puppetlabs/puppetlabs-azure#module-description
