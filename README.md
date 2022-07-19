# SDN-network-project
  SDN network design made with GNS3. The essence of the project was to present the main functionalities of Open vSwitch (OpenFlow, LACP, L3, VLAN).  The materials contained in the repository take the form of screenshots and instructions. Everything you need to recreate the project is included in this file, enjoy :)
  The created infrastructure is shown below:

![topolgia zaprojektowanej sieci](https://user-images.githubusercontent.com/109351514/179216549-021ba692-cc81-47c0-9359-196ee290924c.JPG)

  The structure of the presented network consists of two parts. The first one is located at the top (marked with a yellow line) and represent the controller and application plane. It consists of two virtual machines, Cisco switch, NAT cloud and management interfaces (eth0) belonging to two OVS switches. All the listed items belongs to one subnet (192.168.122.0), where the role of the DHCP server is played by the NAT Cloud. The heart of this plane is the ODL / OFM server, which includes an installed controller (OpenDaylight) and an application (OpenFlow Manager). On the left side is a virtual machine with a simple Linux Lite graphics system, which will facilitate network management using a web browser. 
  The other part is a managed network with two Open vSwitch switches (located in the centre). They are fully synchronized with the controller, unlike the rest of the devices (such as end stations or routers) that setup is mainly done directly. On the left the website has a simple simulated computer (VPCS) and a server containing two NIC's. Both devices are separated from the rest by creating a separate bridge on the switch. They are intended to present an example of the link aggregation  from using the LACP protocol. On the right is another OVS switch that unlike the "colleagues" in the center, it is not connected to controller. Its task is to present another OVS functionality that it's  routing. In the very center, two VLANs were created, in which the composition includes Cisco router, VPCS and Ubuntu Docker Guest (light server).
  Both planes must be separated from each other, therefore on both switches there is no routing configured. This is because, interfaces eth0 of the switches are management-only interfaces. If we however, they ignored this fact, there would be collision of TCP frames with OpenFlow frames, which paralyzes remote management of the SDN network. The switches therefore work in the second layer. In a situation where any of the devices needed internet connection, user can use R5 router which supports NAT translations. Consequently, the guest systems do not have to be on the same subnets as the NAT cloud, it can use a defined pool of public addresses.


INSTALATION AND CONFIGURATION


First, we need to install a virtualization environment (VirtualBox or VMware Workstation Pro) and GNS3 with GNS3 VM. In my case, all machines were run on VirtualBox (ODL/OFM server, LinuxLite, Debian LACP) with the exception of GNS3 VM, which was run on VMware. The rest of devices such as Cisco routers (7200), switches (OVS, IOU), VPCSs and Dockers will be hosted by GNS3 VM. To download the tools mentioned above, visit https://gns3.com/marketplace/appliances and import them into your project.
Then you need to address all devices and connect them with each other as shown in the picture above. To do this, use the console. For example:

![router](https://user-images.githubusercontent.com/109351514/179244962-ef5cab2c-e361-43f6-b10a-6e874b4cd917.JPG)
The next step is to configure the routing tables of the routers. Example:

![ip route](https://user-images.githubusercontent.com/109351514/179245749-4eccf9c4-c5a2-4dac-a2a7-541a746c05a0.JPG)

To allow machines on different subnets to communicate with the Internet, you need to plug the NAT cloud into one of the R5 interfaces. The implementation of this communication requires the definition of a pool of global addresses, so that devices do not have to accept addresses from the subnet to which the NAT cloud belongs:

![PULA NAT](https://user-images.githubusercontent.com/109351514/179246471-ca741aaa-eb2a-44e1-b324-2313efa57dad.JPG)

First, we define which interface will be the input and which will be the output. Then we create an ACL containing a set of source addresses, in our case the "any" option was selected, which allows any addresses. In the next line, the pool of global addresses included in the NAT cloud subnetwork, i.e. 192.168.122.0/24, was configured. The last command activates the translation of the source addresses from the list "1" into the pool of global addresses

Finally, we are getting closer to the heart of the designed network, of course, we are talking about two Open vSwitch placed in the very center of the topology. Before turning on both devices, select the device with the right mouse button and enter the edit config tab. In this tab we need to set the eth0 interface (the management interface) to work in DHCP mode. This interface should receive an address straight from the NAT cloud so that it is on the same subnet as the yet undescribed controller and Linux Lite computer. After saving the initial configuration, we run the switches and turn on the STP (spanning tree protocol) on them, which is to prevent the formation of loops.

![stp](https://user-images.githubusercontent.com/109351514/179247663-b1c4258d-a692-495c-b0c6-6de53b299165.JPG)

Enabling the STP protocol applies to bridge number zero, because by default all switch interfaces (except eth0) are in bridge number zero.
The next necessary step is to create a connection between the switch and the controller, but we'll learn about that in a moment.

Before we pair the switches with the controller, we will start the prepared server with Debian 11 and install the appropriate components. First, we need to import our virtual machine to GNS. After transferring the server to the board, connect it to the network (controller plane) and set the DHCP mode in the / etc / network / interfaces file. After receiving the address, we first install the OpenDaylight controller. Below is a list of the necessary commands:

![ODL installation](https://user-images.githubusercontent.com/109351514/179250197-d95cc046-3e1a-47a5-a8af-a4ddc59fc55d.JPG)
The first five commands update the server, provide abstractions, and add repositories to the package manager. This is because Debian 11 does not have a repository for installing Java version 8. Next, an environment variable is set to be passed to the appropriate child processes. From the eighth to eleventh lines, we focus on installing tools that allow you to download and unpack the directory containing the controller. Enter the unpacked folder and use the command ./bin/karaf to run our container containing the ODL. The remaining commands are used to install the appropriate components that must be on the controller so that it can fully communicate with the switches. In order to better understand the operation of the SDN controller, visit https://www.opendaylight.org/about/platform-overview.

To fully build the architecture of the SDN network, we need an application that will control the flows in the OVS switch tables. To install OpenFlow Manager, follow these steps (on the same machine):

![OFM install](https://user-images.githubusercontent.com/109351514/179251943-58044440-3480-4140-bffd-32dd62bdec8e.JPG)

First, we install the package system, which will be needed for the proper functioning of the nodejs environment, necessary for the proper functioning of the application. Then install git to download its repository containing OpenFlow Manager. After downloading, we need to enter the directory and make changes to the env.module.js file. In the base url line, enter the IP address of our server in order to establish a connection with the controller. By issuing the grunt command, we run the application. Now all you need to do is connect our server with OVS switches. To do this, we need to go back to the switch console for a moment and enter the command in each of them: ovs-vsctl set-controller br0 tcp: [server IP address]: 6633. In our case, the server took the address: 192.168.122.122. Now all you need to do is turn on the Linux Lite virtual machine, set the DHCP mode from the control panel and turn on the browser. To connect to the graphic interface (DLUX), built into the controller, enter: 192.168.122.122:8181/index.html.

![topolgia dlux](https://user-images.githubusercontent.com/109351514/179253047-1aefbc3a-740b-4406-ad6e-32e607b9cf20.JPG)

Before we connect to the controller, it is worth generating network traffic that will refresh the list of displayed hosts in the DLUX program. After performing this action, refresh the topology with the reload button. It is worth remembering because adding a new device to the topology and not reloading it means that the OFM application will not see the newly added devices. Access to the application from the browser level is made by entering: 192.168.122.122:9000.

![topologia OFM](https://user-images.githubusercontent.com/109351514/179253307-dfcb9b5e-eb52-4e38-a2d0-90d5b6332b28.JPG)

Looking to the left side of our topology, we can see two machines, one of which is a server and the other a simulated simple computer. Both devices, as mentioned before, are to be used to test link aggregation using the LACP protocol. VPCS is easily configured from the console level (ip [network address] [mask] [default gateway]). It will be different in the context of the Debian LACP server. Here we have to define two network adapters in VirtualBox itself (enp0s3 and enp0s8). Now we can move on to creating the bond. To do this, we download an additional package with the command: apt-get install ifenslave, then add our bond in the modules directory: echo 'bonding' >> / etc / modules, and finally load the module with the command: modprobe bonding. Now all you have to do is define the bond in the configuration file:

![LACP interfaces](https://user-images.githubusercontent.com/109351514/179254896-f39b1c62-5988-42d7-84b4-acd540f7482b.JPG)

We can see that when defining both cards we deal with the term manual, which is used when the network interface is not to have an IP address. Only now the bond has been addressed statically. Additionally, we can note the type of bonding is marked with the number 4. This number represents the aggregation of dynamic links (LACP). It is important that the switch connected to the server also supports the 802.3ad standard.

We also need to perform some operations on the switch

![LACP OVS](https://user-images.githubusercontent.com/109351514/179256118-80185ede-89e4-46c9-aca8-cb0a0e826b41.JPG)

First, we created a new lacp bridge, then removed three interfaces from the default bridge. Two of them took part in the creation of the bond. A simple simulated computer is connected to the sixth interface, which will allow us to test the bonding.

Remaining in the central switches, VLANs with a trunk were defined, which will allow the different VLANs to communicate. This link is used in a situation where we want to transport packets of several VLANs in one link. To split into several logical networks, the following command was used: ovs-vsctl set port [interface name] tag = [VLAN identifier]. This is how VLAN10 and VLAN20 were configured on both switches. The trunk link was implemented with the command: ovs-vsctl set port [interface name] trunks = [first VLAN ID, second VLAN ID].

The last configuration is to prepare the third OVS switch, not connected to the controller, to work in the third layer. To do this, use the edit tab and define the addresses of connected interfaces there. The next step is to turn on the switch and complete the routing table. For this purpose, a short script was written:

![Routing skrypt OVSL3](https://user-images.githubusercontent.com/109351514/179263724-92b5a583-c410-404c-b73f-7c0d4128746f.JPG)


TESTING


In the figure below. we are in the server console (Docker1), which is in the VLAN10 network. We can see that the connection was established only with the networks: 192.168.50.0 and 192.168.60.0, which confirms the correct operation of not only logical segments, but also the trunk link. No communication could be established with the other networks. Thanks to this solution, we can confidently isolate selected networks from each other and more:

![VLAN ping](https://user-images.githubusercontent.com/109351514/179264756-6a0eaba8-0656-4b85-b524-88e1f5179b26.JPG)

Now let's test the configured link aggregation using LACP. According to our requirements, we should obtain two functionalities: transferring traffic to the available link in the event of breaking the current connection and increasing the transmission speed by using all available connections. To verify the correctness of operation, the Wireshark program was used, which allows intercepting the traffic on a given link:

![LACP WIRESHARKprzerzucenie ruchu](https://user-images.githubusercontent.com/109351514/179265156-cf517df5-6629-4836-9b4b-bc689ed20665.JPG)

After a short while, the network traffic is transferred to the available link. When we re-plug the virtual cable between the eth4 (switch) and e1 (DebianLACP) interface, we will notice that ICMP queries and responses are grouping together. Only the answers are sent on one link, and only the queries on the other. The described situation is presented in the figure below:

![LACP wireshark dwa kanały](https://user-images.githubusercontent.com/109351514/179266772-1dc04249-3926-4844-b880-0a196a0f3754.JPG)

The last described functionality will be remote management of switches using an application connected to the controller. Information on flows using the OpenFlow protocol will be sent on virtual links connected to management interfaces. With this feature, it is safe to say that the designed network is an SDN programmable network.
The first step will be to start the server and install the tmux program, which will allow you to run the controller and the application at the same time. To split the screen horizontally, press: "Ctrl + Shift + B +":

![ODL i OFM włączenie](https://user-images.githubusercontent.com/109351514/179269052-886f2384-2485-428a-8e7c-4242f969f0c0.JPG)

