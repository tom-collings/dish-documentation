The initial request for a Pivotal Cloud Foundry (PCF) install was to include four tiles:  OpsManager (version 1.5.0), Elastic Runtime (version 1.5.1), OpsMetrics (version 1.4) and Spring Cloud (beta tile, version 0.1).  The Spring Cloud tile contains dependencies on the RabbitMQ (version X.X) and MySQL (version X.X) tiles.

A consideration for installing PCF is that two networks are recommended for a proper install.  These networks are named sys.<network suffix> and apps.<network suffix> where network suffix indicates the environment in which PCF is being deployed.  The network suffix follows this convention:

<name>.<environment>.<data center>.cfdish.io

where <name> would be in <INT, DMZ, CON, BUS, PCI, NONPCI> reflecting internal, DMZ, Consumer, Business, PCI-compliant and Non-PCI-Compliant, respectively.

and <environment> would be in <DEV, TEST, PROD>

and <data center> would be in <MER, CHY>

Because PCF, under the covers, generates DNS names for VMs it creates for its use, we are required to have wildcard DNS entries for *.apps.<network suffix> and *.sys.<network suffix>.  Further, because all communication is to be over SSL, we will also require wildcard certificates for *.apps.<network suffix> and *.sys.<network suffix>.

The issue of using Symantec-signed certificates arose.  Because we are installing in a total of twelve environments, and we are required to have two wildcard certificates per environment, we were looking at a total of 24 Symantec-signed wildcard certificates.  This would have been an unexpected cost to the program.  

After consideration, we realized that the certs for the *.sys.<suffix> domains are only accessed internally, and can use the Dish CA cert as its authority.  That removed the necessity for twelve of the Symantec-signed certs.  Further, we could use Dish CA certs for the non-production environments, with the only risk being a brownfield app accessing a new cloud foundry app (which is unlikely, as there are not any PCF apps at this point).  To test the effectivity of a Dish CA-Signed certificate, we are starting with the INT-TEST and INT-DEV environments.  If those work well, we can consider using Dish CA-signed certificates in all environments, and protect the applications with vanity names, using Symantec-signed certificates there.

While deploying the MySQL tile, we discovered that a part of the installation for that tile is retrieving configuration from the Cloud Controller VM, but in doing so is going back out through the F5 to get to that VM (as opposed to going directly to the VM).  It is attempting the retrieve that configuration via HTTP, which means that we need to have, at least for the *.sys.<suffix> traffic on the F5, port 80 open to allow for this traffic.  Port 443 should be terminating the SSL connection using the certificates described above, and redirecting to port 80.

As we were attempting to install and configure the Spring Cloud tile, we discovered that it had not been certified for use in a 1.5.x OpsManager environment, and the recommendation was to revert back to a 1.4.x version of OpsManager.  We deleted our 1.5 instances in the DEV and TEST environments and began installing and configuring a 1.4.2 environment for both DEV and TEST.

Here we discovered that the push-console errand (run as a function of the Elastic Runtime configuration for version 1.4) requires a communication between the login and uaa components.  These components go out to the F5 to route this communication, but are expecting to use the certificate we self-generated as a part of the Elastic Runtime configuration (as opposed to the Dish CA-signed certificate installed on the F5).  This left us with two choices:

1.  Install our self-signed certificate on the F5, or
2.  Copy and paste the certificate and key installed on the F5 down to the Elastic Runtime configuration.

Neither of these solutions was acceptable to the F5 team - completely self signed certificates on the F5 were not an option from a security perspective, and leaving the Dish CA-signed certs available in Cloud Foundry meant more exposure of those certs as well as more maintenance required for the team managing them for Dish.  A further discussion on this point led us to discover that the Spring Cloud tile was not a requirement after all - that we could continue with the install and configuration of a 1.5.0 environment and install the Spring Cloud tile once it had been certified by Pivotal.
