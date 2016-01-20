---
layout: page
title: Roadmap
---

This section records high-level the plan of record, but is typically
less detailed than information on the
[Development Mailing List](https://groups.google.com/a/xosproject.org/forum/#!forum/devel).
Also see bug reports and short-term feature requests filed at the   [GitHub Issue
Tracker](https://github.com/open-cloud/xos/issues).

##Cozad Release (December 2015)

1. Separate subsystems into their own containers. For example,
    XOS and Postgress should run in separate containers. Synchronizers
    should also be isolated in their own containers.  We also need to do
    some design work around how to best extend the data model schema.

2. Address XOS scalability and high availability. Both are a function
   of the underlying database solution.

3. Leverage OpenStack domains so XOS can run on autonomous
    OpenStack clusters.

4. Generalize Compute Instances to include both VMs and Containers.

5. Extend support for "subscribers" (and other "service users") by
    taking advantage of external identity management systems; e.g.,
    via OpenID. 

6. Related to (5), revisit the XOS data model for users, roles, and
    privileges.

7. Improve UI programming environment, including (a) re-engineering
    xoslib to support Single Page Application, (b) re-factoring
    to support a Js StyleGuide, and (c) providing a view-generation
	tool.

8. Provide a complete set of views, including
   * Operator: Complete "XOS Admin", Nagios, Horizon, ELK-Stack.
   * Developer: Evolve "Tenant View" into "Developer View".
   * Service User: Content Provider (CDN), Syndicate, CORD Subscriber.
   * *[Note: Current "Developer View" conflates Operator, Developer, 
     and Subscriber perspectives. Re-factor as outlined above.]*

9. Integrate Monitoring Service into XOS. The plan is to make
    OpenStack's Ceilometer a first-class XOS service, including
   instrumentation of all software services.

10. Integrate ONOS into XOS. The plan is to make ONOS a
   first-class XOS service, including replacing the default
   OpenStack/Neutron networking support with ONOS.

11. Define a data model migration strategy. The current strategy
    too often depends on resetting the data model.

##Deployment Configurations (Oct - Dec 2015)

XOS can be deployed on different hardware configurations and populated
with different service portfolios. We expect the following deployment
configurations to be based on the Burwell release.

1. Development: A minimal configuration used for development, with backend 
cloud resources provided by CloudLab. Corresponds to the environment 
described in [Developer Guide](/devguide/). Available with the 
Burwell release.

2. Test: A configuration used used to run regression tests on the XOS
core, with backend cloud resources provided by CloudLab. Available
with the Burwell release.

3. CORD Development: A configuration used for CORD development. 
Includes vSG, ONOS, and minimal vOLT and vRouter. Leverages CloudLab 
servers rather than actual POD hardware. Avaiable by end of October 2015. 

4. OpenCloud: Deployed across multiple Internet2 sites. Service
portfolio includes OpenStack, EC2, Syndicate, and vCDN. Available by
end of November 2015.

5. CORD POD: A configuration to be deployed on a CORD POD with OLT
hardware support. Service portfolio includes OpenStack, ONOS,
Ceilometer, vCDN, vOLT, vSG, and vRouter. Avaiable by end of December 2015.
