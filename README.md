--- Secure Azure Landing Zone ---

This is a project I built to get real, hands-on experience with Azure while studying for the AZ-104 (Azure Administrator) certification. Reading about cloud concepts only gets you so far so I wanted to actually build a secure environment from scratch and run into the real problems that come with it.

The idea was simple: instead of spinning up resources one at a time and worrying about security later, I'd lay down a proper foundation first: networking, identity, governance, and storage, all secured from the start and then everything built on top of it would inherit those good practices automatically. That's what a "landing zone" is. Mine is a scaled-down, single-subscription version of the pattern Microsoft uses in its Cloud Adoption Framework.

Author: Max Xing
Region: East US
Status: In progress - identity, governance, networking, and storage are done. compute and monitoring are next.

-- The main idea: Least Privilege
Only ever grant the minimum access needed, and deny everything else by default. I tried to apply that same principle at every layer:

Identity - roles grant the minimum permissions needed, scoped as narrowly as possible
Governance - policies allow only whats permitted (specific regions, required tags)
Network - firewalls deny by default and allow only the traffic that has to flow
Storage - isolated from the public internet, encrypted everywhere, no anonymous access

The point is defense-in-depth: every layer assumes the others might fail, so there's no single thing that, if it breaks, leaves everything exposed.

-- My naming convention: <resource-type>-<workload>-<environment>-<region>-<instance>

--How I tagged things --

Every resource carries the same set of tags so I can track cost and enforce governance:

Environment = Dev, Project = LandingZone, Owner = Max Xing, ...

-- Identity and Governance -- 

This is where I started, and it's the foundation for everything else.

Resource group - I created rg-landingzone-dev-cc-001 to hold the whole project, with the full tagging strategy applied.

A custom RBAC role - I built a custom role called "VM Operator" to really understand how permissions work. It can read, start, restart, deallocate, and power off virtual machines but it can't create, modify, or delete them.

Group-based access - Instead of assigning that role to a person directly, I created a VM-Operators security group, gave the role to the group, and added a test user to it. When someone joins or leaves a team, you change group membership, not individual role assignments. You can see the difference clearly in the access list: I have broad Owner access inherited from the subscription, while the operator group has narrow access scoped to just this resource group.

Azure Policy - I assigned two policies to enforce rules automatically rather than relying on anyone to remember them: one that restricts deployments to a specific region and one that requires an Environment tag on every resource. The second one actually rejected a resource I created without the tag which is good because it meant the guardrail was working, and I got to troubleshoot a real policy denial.

Resource locks - I put a CanNotDelete lock on the resource group to prevent accidental deletion.

-- Networking -- 

With the foundation in place, I built the network the resources would live in.

Virtual network - vnet-landingzone-dev-eus-001 with an address space of 10.0.0.0/16, giving me plenty of room to carve into subnets.

Two subnets - I split the network into a web tier (snet-web-001, 10.0.1.0/24) for anything internet-facing, and an app tier (snet-app-001, 10.0.2.0/24) for backend resources that should stay hidden.

-- Network Security --

Network Security Groups - I created two firewalls. The web NSG allows inbound HTTPS from the internet and nothing else. The app NSG only accepts traffic from the web subnet and explicitly denies the internet. So the backend is reachable only through the front end, never directly from outside.

Application Security Groups - I created asg-web and asg-app so that once VMs are deployed, I can write NSG rules against logical roles ("allow web → app") instead of hardcoded IP addresses that break whenever the network changes.

A User Defined Route - I added a route on the app subnet that drops all internet-bound traffic (next hop set to None). So the backend is protected three different ways: the NSG limits where traffic can come from, an explicit rule denies the internet, and the route table won't even let traffic leave.

-- Storage --

Storage account - I deployed salandingzonedeveus001 (Standard, Locally-Redundant Storage) and locked it down: encryption at rest is on by default, secure transfer forces HTTPS, the minimum TLS version is 1.2, and anonymous access is off.

Private container - I created a data container with no anonymous access.

Network isolation - This is the part I'm most happy with. I restricted the storage account's network access to my VNet using a service endpoint, which means it's no longer reachable from the public internet at all, only from inside my network. Even if someone got hold of the storage keys, they couldn't use them from outside.

-- Whats Next --
Compute - Deploy a Linux VM in the web subnet and a Windows VM in the app subnet, drop them behind the NSGs and into the ASGs I already set up, and wire up the managed identity for storage access.

Monitoring & protection - Turn on Azure Monitor, Microsoft Defender for Cloud, and backup/alerting to round out the environment.

-- Takeaways --
Again, this project is built on the idea of least privilege so the idea of only allowing whats needed ran from RBAC to NSGs to storage networking. This made the entire project more consistent as it hit the goal of being a secure landing zone.

I also mentioned my own policy rejecting me. This helped me learn more about tuning policy parameters than reading about them in the Azure course. When I made them at first it still felt a little abstract but once I dug deeper to solve the issues, it felt more concrete. 
This actually happened twice but the second time was a built in region-restriction policy above my subscription and I worked around it by depoloying my resources in East US, different from Canada Central where my resource group lives.
