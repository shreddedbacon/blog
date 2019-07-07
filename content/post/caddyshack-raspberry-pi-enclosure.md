---
title: "Caddyshack - RaspberryPi Blade Chassis"
date: 2019-07-07T21:40:00+11:00
---

I've been working on another project when I get motivated to do so, the project is a RaspberryPi enclosure set that turns RaspberryPis or other single board computers into a sort of blade chassis. Part of this project is to sort of have an openstack like aproach to managing it. Sort of like a rPiaas (RaspberryPi as a Service) with an IaaS component.

<!--more-->
## Caddyshack
Caddyshack is what it is called. As the enclosure is a shack, and it holds caddies.

The idea is that the enclosure contains a backplane, which allows remote powering on and off of individual caddies, but also allows for USB passthrough for accessing the RaspberryPi onboard serial interface for remote access if required.

Each caddy is fitted with a passthrough board that connects it to the backplane, this is how power and usb is passed through from the backplane.

Each backplane unit can operate 3 or 4 caddies depending on configuration, and contains a WemosD1 mini running MQTT PubSub to subscribe and publish to different topics per caddy. And a single enclosure can fit 13 caddies (3x3Caddy BPU and 1x4Caddy BPU)

There is a main RaspberryPi that is not in a caddy, but runs on the rear of an enclosure that runs the primary software.

Other things can also go into a caddy, like a cheap 5 port switch, or a small router, or an odroid HC1 with an SSD.

## Hardware Components
There are a few components that make up a Caddyshack.

* Backplane PCB Unit (for 3 or 4 caddies)
* Passthrough/Control PCB Unit (1 per caddy)
* USB Hub (1 per backplane unit)
* Shift Register Unit (3 or 4 per backplane unit)
* Enclosure itself, 2020 Extrusion and Acrylic or 3d printed panels
* Caddy hardware, full 3d printable

## Software Components
Caddyshack is designed to be a microservice architecture, and I am slowly working towards getting it to that point.

The services are as follows at the moment
* Backplane firmware
  * This runs arduino and connects to MQTT and interacts with the individual caddies based on pubsub messaging from the Controller
* Controller
  * This runs on the main controller RaspberryPi. It is the gateway to all the other parts of caddyshack, the primary interface that a user will see (or use for the API)
* Remote Consoles
  * This runs on the RaspberryPi that will connect to the USB hubs on each backplane unit, and this allows access to the serial interface on each caddy, and provides a `shellinabox` session to communicate directly.
  * The Controller can talk to this service to retrieve the URL for the `shellinabox` sessions to provide to the user.
* Operating System NFS Server
  * This runs on something that should have physical storage like an SSD, with enough network to run an NFS server that will provide mount points for NFS booting RaspberryPis
  * The Controller can talk to this service to allow a Pi to have its operating system change.

## Where it's up to
Still hugely a prototype that is failing.

I have had to redesign all the PCBs due to a silly error, but haven't yet sent them off to be manufactured. But through limited testing with old boards that were modified they should be good.

I also need to finish writing the services, but a lot of the testing relies on working hardware (to test USB serial comms etc)

One bit that is also missing, is the ability to control networks. I have the main bits sorted, being able to change the operating system, and use remote console, but there is no way to currently isolate Pis via VLANs. But I am working on something else at the moment that allows cheap dumb switches that have the right chips on them to be manageable (vlan access ports, tagged trunk ports) and writing an API that the Controller will be able to interact with.
