---
title: "Eudyptula Task5"
date: 2022-06-22T16:14:27+05:30
draft: false
# showtoc: false
tags: [Eudyptula, linux]
series: [Eudyptula]
description: Task 5 for Eudyptula challenge
# math: true
# ShowBreadCrumbs: false
# searchHidden: true
# hideMeta: true
---


```txt{linenos=false}
This is Task 05 of the Eudyptula Challenge
------------------------------------------

Yeah, you survived the coding style mess!  Now, on to some "real"
things, as I know you are getting bored by these so far.

So, simple task this time around:
  - take the kernel module you wrote for task 01, and modify it so that
    when a USB keyboard is plugged in, the module will be automatically
    loaded by the correct userspace hotplug tools (which are implemented
    by depmod / kmod / udev / mdev / systemd, depending on what distro
    you are using.)

Yes, so simple, and yet, it's a bit tricky.  As a hint, go read chapter
14 of the book, "Linux Device Drivers, 3rd edition."  Don't worry, it's
free, and online, no need to go buy anything.
```


### What is USB??

Ofcourse, we know what a USB is!! We use it everyday with our digital devices like pen-drive, external harddisks, chargers, digital camera, keyboard, mice, wifi dongle, etc...USB devices and connectors are popular with everyone, even with people without any IT background. It is so famous and well known that it has got [it's own website](https://www.usb.org/) [^1] and a [Wikipedia page](https://en.wikipedia.org/wiki/USB) [^2].
[^1]: https://en.wikipedia.org/wiki/USB
[^2]: https://www.usb.org/

In early days, just before USB appeared, peripherals like keyboards, mouse and printers were connected with serial and parallel ports. Problem with that was if you accidentally stuck a mouse into the socket for keyboard, it won't work. And that's not it, once you successfully connect the device in the correct socket..you now need to install the proper driver for it.

If all went well, a quick reboot after the driver install and you'll have your working device ready to be used. Naturally, people would want something better and easy than this - "One port to rule them all"


![usb](https://media.giphy.com/media/3ov9jNkYm8QqxakeCQ/giphy.gif#center)

And soon, the USB (Universal Serial Bus) was born as a replacement for the serial, parallel and PS/2 ports.

The USB specification went on to have several revisions, with the major ones being 2.0 in 2001, 3.0 in 2008, and the very latest spec (4.0) released in 2019. Let's take a look at how USBs work.

### USB in linux

Linux kernel has a dedicated sub-system to handle USB, you can read everything about it in detail from [here](http://www.linux-usb.org/USB-guide/book1.html) [^3]
[^3]: http://www.linux-usb.org/USB-guide/book1.html

Let me start with a terminal command to begin explaining things. There are utilities that can help you to identify the system hardware...you can find any device connected to your system, more specifically you can get the USBs attached with `lsusb` command

```txt{linenos=false}
# COMMAND ->   lsusb --tree --verbose

/:  Bus 01.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/6p, 5000M
    ID PP01:VV02 Linux Foundation 3.0 root hub


/:  Bus 02.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/12p, 480M
    ID PP01:VV01 Linux Foundation 2.0 root hub
    |__ Port 1: Dev 2, If 1, Class=Human Interface Device, Driver=usbhid, 12M
        ID 046d:c534 Logitech, Inc. Unifying Receiver
    
    |__ Port 1: Dev 2, If 0, Class=Human Interface Device, Driver=usbhid, 12M
        ID 046d:c534 Logitech, Inc. Unifying Receiver
    
    |__ Port 5: Dev 3, If 1, Class=Wireless, Driver=btusb, 12M
        ID PP03:VV03 Lite-On Technology Corp. Qualcomm Atheros QCA9377 Bluetooth
    
    |__ Port 5: Dev 3, If 0, Class=Wireless, Driver=btusb, 12M
        ID PP03:VV03 Lite-On Technology Corp. Qualcomm Atheros QCA9377 Bluetooth
    
    |__ Port 7: Dev 4, If 0, Class=Video, Driver=uvcvideo, 480M
        ID PP04:VV04 Realtek Semiconductor Corp.
    
    |__ Port 7: Dev 4, If 1, Class=Video, Driver=uvcvideo, 480M
        ID PP04:VV04 Realtek Semiconductor Corp.
```

This output shows a list of USB host controllers (Bus 02 and Bus 01) and the actual physical devices connected to them. At then end of each line, the negotiated speed limits are shown in Mbits/s. There are different negotiated speeds for different kinds of devices.
- Full speed mode (12 Mbits/s) is used for communicating with keyboard, mice and other similar devices. 
- Hi-Speed mode (480 Mbit/s) is used for communication with storage devices, webcams, and other devices which demand more bandwidth.
- Ports with 5000 Mbits/s (5Gbits/s) is USB 3.0 port.


As the output shows, my laptop has two USB host controllers. One of them is USB3(5000 Mbits/s) and another is USB2(480 Mbits/s). Both of them are managed by `xhci_hcd/6p` and `xhci_hcd/12p` driver respectively. 

The device numbers (Dev 1, Dev2, ...) are just numbers given by kernel to identify each device. If you eject and plug the same USB you might get another device number for it.

Now, let's focus on bus #02. This bus has 3 types/class of devices connected to it:-

1. Human Interface Device (for my keyboard and mice)
2. Wireless (for Bluetooth adapter card)
3. Video (for video camera)


Each USB device contains a vendor ID and a product ID which helps programs to detect the device and load proper driver for it. To clarify, companies pay to acquire Vendor IDs from the USB Implementers Forum. A complete list of vendor IDs with their respective product ID can be found [here](http://www.linux-usb.org/usb.ids) [^4]

[^4]: http://www.linux-usb.org/usb.ids


USB implementation is complex by design. There are tons of things that are abstracted from the regular users... but for a kernel developer working on USB device drivers, it is very useful to know how things work beyond the simple abstraction layer. 


#### More USB info
There is some information that `lsusb` do not provide, to make things easy for end user... We can use `usb-devices` command for that stuff.


```txt{linenos=false}
T:  Bus=01 Lev=00 Prnt=00 Port=00 Cnt=00 Dev#=  1 Spd=480 MxCh=12
D:  Ver= 2.00 Cls=09(hub  ) Sub=00 Prot=01 MxPS=64 #Cfgs=  1
P:  Vendor=1d6b ProdID=0002 Rev=05.18
S:  Manufacturer=Linux 5.18.3-arch1-1 xhci-hcd
S:  Product=xHCI Host Controller
S:  SerialNumber=0000:00:14.0
C:  #Ifs= 1 Cfg#= 1 Atr=e0 MxPwr=0mA
I:  If#= 0 Alt= 0 #EPs= 1 Cls=09(hub  ) Sub=00 Prot=00 Driver=hub
E:  Ad=81(I) Atr=03(Int.) MxPS=   4 Ivl=256ms

T:  Bus=01 Lev=01 Prnt=01 Port=00 Cnt=01 Dev#=  5 Spd=12  MxCh= 0
D:  Ver= 2.00 Cls=00(>ifc ) Sub=00 Prot=00 MxPS= 8 #Cfgs=  1
P:  Vendor=046d ProdID=c534 Rev=29.01
S:  Manufacturer=Logitech
S:  Product=USB Receiver
C:  #Ifs= 2 Cfg#= 1 Atr=a0 MxPwr=98mA
I:  If#= 0 Alt= 0 #EPs= 1 Cls=03(HID  ) Sub=01 Prot=01 Driver=usbhid
E:  Ad=81(I) Atr=03(Int.) MxPS=   8 Ivl=8ms
I:  If#= 1 Alt= 0 #EPs= 1 Cls=03(HID  ) Sub=01 Prot=02 Driver=usbhid
E:  Ad=82(I) Atr=03(Int.) MxPS=  20 Ivl=2ms

T:  Bus=02 Lev=00 Prnt=00 Port=00 Cnt=00 Dev#=  1 Spd=5000 MxCh= 6
D:  Ver= 3.00 Cls=09(hub  ) Sub=00 Prot=03 MxPS= 9 #Cfgs=  1
P:  Vendor=1d6b ProdID=0003 Rev=05.18
S:  Manufacturer=Linux 5.18.3-arch1-1 xhci-hcd
S:  Product=xHCI Host Controller
S:  SerialNumber=0000:00:14.0
C:  #Ifs= 1 Cfg#= 1 Atr=e0 MxPwr=0mA
I:  If#= 0 Alt= 0 #EPs= 1 Cls=09(hub  ) Sub=00 Prot=00 Driver=hub
E:  Ad=81(I) Atr=03(Int.) MxPS=   4 Ivl=256ms
```

This definitely gives more information than the previous command... but it is all alien looking at first glance. Slowly, you can see that few bits are not so alien after all. We can get the `Bus`, `Port`, `Spd`(Speed), `Cls`, `Vendor`, `ProdID` from the above output as we were able to via `lsusb` command.

Now it's time to understand more about USBs and get into the abstraction layer and dig deeper.

Typically, USB devices uses it's own file-system on linux, which is a dynamically generated file-system that complements the normal device node system, and can be used to write user space device drivers. *Just remember that this is not always the case.*

Irrespective of what the case is, you can get above output (or something similar) using multitude of tools/utilities... you can just focus on understanding the output of such tools/utilities for now.

The information in the above output is arranged in groups and each group has 1 character that actually indicates the `type` of the information written in that specific line. See below table for complete list:

| Character | Meaning |
|-----------|---------|
| T | Topology information |
| D | Device Descriptor information|
| P | Similar to D; Product info (fetched from device descriptor info|
| S | String info; returned by device|
| C | Configuration Descriptor information|
| I | Interface Descriptor information|
| E | Endpoint Descriptor information|


To get a complete understanding of the above, we need more knowledge around how USB system works inside linux kernel... Or more specifically, how usb endpoints, Interfaces and configurations all fit together in the topology. 

According to USB protocol (standardized by USB Implementers Forum or USB-IF folks), USB is a *cable bus that supports data exchange between host computer and a wide range of simultaneously accessible peripherals.*


USB protocol uses *star-topology* starting from a single node called "**host**" or "**root-hub**" with branches to other nodes. These nodes could be another level **hub** or simply a functional **peripheral device** (device used by user).

Because of this topology, USB device can never start sending data without being asked first... So it makes USB host controller in-charge of asking every USB device if it has any data to send. This allows for a very easy plug-n-play type of system, where devices can be easily configured by the host computer.

To remove the necessity of special drivers for different kind of devices, USB protocol specifications has defined some set of standards that any device of a specific type can follow. These types are called classes. Complete list of USB specified classes can be found [here](https://www.usb.org/defined-class-codes) [^5] with their class codes.

[^5]: https://www.usb.org/defined-class-codes

The idea of installing drivers for each and every USB device (manually) and then rebooting the system is very scary to me. This type of specification helps a lot. Still if there are some special devices which you need to work with, then you can install the special device driver for that and it'll still work. It means, you get support for most kinds of USB devices by default and you can still add more support without changing anything in the design.

Unfortunately, USB protocol specification is a multi-layer protocol specification which makes USBs a lot more complex and abstracted than we expected. Fortunately, most of this complexity is handled by **USB core subsystem** [^6]. 
[^6]: https://www.kernel.org/doc/html/latest/driver-api/usb/index.html


Linux Device drivers (normally implemented as a loadable kernel modules) have to set up few thing before they can actually start functioning as USB devices. Few of these things involve setting up configurations, interfaces and endpoints; And then binding USB device to USB interface.

#### Endpoints

Most basic form of USB communication is through **endpoints**. Endpoints are like pipes which carry data in a single direction (*a unidirectional pipe*) only, either from my laptop to device (**OUT endpoint**) or from the USB device to my laptop (**IN endpoint**).

Each endpoint has an associated **DescrptorType** to it which defines how the data is transmitted through that endpoint. There can be 4 different type of such descriptors:-

1. CONTROL
    - Normally used to configure, retrieve info, send info or get status reports about the device.
    - Every USB has a control endpoint with associated number = 0

2. INTERRUPT
    - transfers small amounts of data at fixed rate every time somebody asks device for data.
    - Usually used with keyboard and mice; anywhere we need to send a signal to device using buttons or similar methods.

3. BULK
    - Transfer larger amounts to data with no data loss.
    - Common for printer, storage and network devices.

4. ISOCHRONOUS
    - Also transfer large amounts of data, but data loss is possible.
    - Usually for real time devices or any constant streaming data.
    - Commonly found in audio and video devices.

In linux kernel, this is implemented using `struct usb_host_endpoint` [^7]
[^7]: https://elixir.bootlin.com/linux/latest/source/include/linux/usb.h#L67


```c
struct usb_host_endpoint {
	struct usb_endpoint_descriptor		desc;
	struct usb_ss_ep_comp_descriptor	ss_ep_comp;
	struct usb_ssp_isoc_ep_comp_descriptor	ssp_isoc_ep_comp;
	struct list_head		urb_list;
	void				*hcpriv;
	struct ep_device		*ep_dev;	/* For sysfs info */

	unsigned char *extra;   /* Extra descriptors */
	int extralen;
	int enabled;
	int streams;
};
```

Each endpoint has their own descriptor attached to it, which is defined with `struct usb_endpoint_descriptor` [^8] . This structure contains the actual information provided by the device.
[^8]: https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/usb/ch9.h#L407

```c
/* USB_DT_ENDPOINT: Endpoint descriptor */
struct usb_endpoint_descriptor {
	__u8  bLength;
	__u8  bDescriptorType;

	__u8  bEndpointAddress;
	__u8  bmAttributes;
	__le16 wMaxPacketSize;
	__u8  bInterval;

	/* NOTE:  these two are _only_ in audio endpoints. */
	/* use USB_DT_ENDPOINT*_SIZE in bLength, not sizeof. */
	__u8  bRefresh;
	__u8  bSynchAddress;
} __attribute__ ((packed));
```

We can already see that there is a `bDescriptorType` variable in the `struct usb_endpoint_descriptor`. This is what defines the type of the endpoint - either *Control*, *Interrupt*, *Bulk* or *Isochronous*. Along with this, this field also contains the direction of the endpoint - either *IN* or *OUT*. The bit-masks `USB_DIR_OUT` and `USB_DIR_IN` can be placed against this field to determine the direction of the endpoint.


#### Interfaces

Zero or more of such endpoints are bundled up into an **Interface**. These interfaces handle only one type of USB connection - such as mouse, touch-pad, keyboard, storage, video stream, etc..
USB interfaces may have alternate settings, which are different choices for parameters of the interface. In kernel, this is implemented as `struct usb_interface` [^9] .

[^9]: https://elixir.bootlin.com/linux/latest/source/include/linux/usb.h#L232

```c
struct usb_interface {
	/* array of alternate settings for this interface,
	 * stored in no particular order */
	struct usb_host_interface *altsetting;

	struct usb_host_interface *cur_altsetting;	/* the currently
					 * active alternate setting */
	unsigned num_altsetting;	/* number of alternate settings */

    int minor;			/* minor number this interface is
					 * bound to */

    ... snip snip ...

};
```

`struct usb_interface` is the structure which USB core passes to USB drivers and what the USB driver is then in charge of. Each `usb_interface` can have multiple settings, but only one setting will be used at a point in time. These settings are defined in another struct `struct usb_host_interface`. [^10]
[^10]: https://elixir.bootlin.com/linux/latest/source/include/linux/usb.h#L82

```c
struct usb_host_interface {
	struct usb_interface_descriptor	desc;

	int extralen;
	unsigned char *extra;   /* Extra descriptors */

	/* array of desc.bNumEndpoints endpoints associated with this
	 * interface setting.  these will be in no particular order.
	 */
	struct usb_host_endpoint *endpoint;

	char *string;		/* iInterface string, if present */
};
```

`struct usb_host_interface` contains a `struct usb_host_endpoint` which is the USB endpoint structure we discussed above.



#### Configurations

One or more USB interfaces are themselves bundled in a USB configuration. A USB device can have multiple configurations and might switch between them in order to change the state of the device.

In linux kernel, It is defined as `struct usb_host_config` [^11] .

[^11]: https://elixir.bootlin.com/linux/latest/source/include/linux/usb.h#L374


```c
struct usb_host_config {
	struct usb_config_descriptor	desc;

	char *string;		/* iConfiguration string, if present */

	/* List of any Interface Association Descriptors in this
	 * configuration. */
	struct usb_interface_assoc_descriptor *intf_assoc[USB_MAXIADS];

	/* the interfaces associated with this configuration,
	 * stored in no particular order */
	struct usb_interface *interface[USB_MAXINTERFACES];

	/* Interface information available even when this is not the
	 * active configuration */
	struct usb_interface_cache *intf_cache[USB_MAXINTERFACES];

	unsigned char *extra;   /* Extra descriptors */
	int extralen;
};
```

Linux defines the USB configurations as above `struct usb_host_config` and the entire USB device as `struct usb_device` [^12]

[^12]: https://elixir.bootlin.com/linux/latest/source/include/linux/usb.h#L626

```goat

┌────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                                                                │
│                                                                                                                                │
│      Device                                                                                                                    │
│                                                                                                                                │
│                                                                                                                                │
│          ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────┐           │
│          │                                                                                                         │           │
│          │    CONFIG 1                                                                                             │           │
│          │                                                                                                         │           │
│          │  ┌──────────────────────────────┐   ┌──────────────────────────────┐   ┌──────────────────────────────┐ │           │
│          │  │                              │   │                              │   │                              │ │           │
│          │  │    Interface 1               │   │    Interface 2               │   │  Interface 3                 │ │           │
│          │  │                              │   │                              │   │                              │ │           │
│          │  │    ┌────────────────────┐    │   │    ┌────────────────────┐    │   │    ┌────────────────────┐    │ │           │
│          │  │    │                    │    │   │    │                    │    │   │    │                    │    │ │           │
│          │  │    │   ENDPOINT 1       │    │   │    │   ENDPOINT 2       │    │   │    │   ENDPOINT 3       │    │ │           │
│          │  │    │                    │    │   │    │                    │    │   │    │                    │    │ │           │
│          │  │    └────────────────────┘    │   │    └────────────────────┘    │   │    └────────────────────┘    │ │           │
│          │  │                              │   │                              │   │                              │ │           │
│          │  │    ┌────────────────────┐    │   │    ┌────────────────────┐    │   │    ┌────────────────────┐    │ │           │
│          │  │    │                    │    │   │    │                    │    │   │    │                    │    │ │           │
│          │  │    │   ENDPOINT 1       │    │   │    │   ENDPOINT 2       │    │   │    │   ENDPOINT 3       │    │ │           │
│          │  │    │                    │    │   │    │                    │    │   │    │                    │    │ │           │
│          │  │    └────────────────────┘    │   │    └────────────────────┘    │   │    └────────────────────┘    │ │           │
│          │  │                              │   │                              │   │                              │ │           │
│          │  └──────────────────────────────┘   └──────────────────────────────┘   └──────────────────────────────┘ │           │
│          │                                                                                                         │           │
│          └─────────────────────────────────────────────────────────────────────────────────────────────────────────┘           │
│                                                                                                                                │
│                                                                                                                                │
│          ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────┐           │
│          │                                                                                                         │           │
│          │    CONFIG 2                                                                                             │           │
│          │                                                                                                         │           │
│          │  ┌──────────────────────────────┐   ┌──────────────────────────────┐   ┌──────────────────────────────┐ │           │
│          │  │                              │   │                              │   │                              │ │           │
│          │  │    Interface 1               │   │    Interface 2               │   │  Interface 3                 │ │           │
│          │  │                              │   │                              │   │                              │ │           │
│          │  │    ┌────────────────────┐    │   │    ┌────────────────────┐    │   │    ┌────────────────────┐    │ │           │
│          │  │    │                    │    │   │    │                    │    │   │    │                    │    │ │           │
│          │  │    │   ENDPOINT 1       │    │   │    │   ENDPOINT 2       │    │   │    │   ENDPOINT 3       │    │ │           │
│          │  │    │                    │    │   │    │                    │    │   │    │                    │    │ │           │
│          │  │    └────────────────────┘    │   │    └────────────────────┘    │   │    └────────────────────┘    │ │           │
│          │  │                              │   │                              │   │                              │ │           │
│          │  │    ┌────────────────────┐    │   │    ┌────────────────────┐    │   │    ┌────────────────────┐    │ │           │
│          │  │    │                    │    │   │    │                    │    │   │    │                    │    │ │           │
│          │  │    │   ENDPOINT 1       │    │   │    │   ENDPOINT 2       │    │   │    │   ENDPOINT 3       │    │ │           │
│          │  │    │                    │    │   │    │                    │    │   │    │                    │    │ │           │
│          │  │    └────────────────────┘    │   │    └────────────────────┘    │   │    └────────────────────┘    │ │           │
│          │  │                              │   │                              │   │                              │ │           │
│          │  └──────────────────────────────┘   └──────────────────────────────┘   └──────────────────────────────┘ │           │
│          │                                                                                                         │           │
│          └─────────────────────────────────────────────────────────────────────────────────────────────────────────┘           │
│                                                                                                                                │
│                                                                                                                                │
│                                                                                                                                │
└────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

- Relationship layout of endpoints, interfaces and configurations with device
```

So to summarize, 
- Devices usually have 1 or more configurations.
- Configs have 1 or more interfaces.
- Interfaces have 1 or more settings.
- Interfaces have 0 or more endpoints.


#### Understanding output of usb-devices

Looking back at the output of `usb-devices` command, with the newly gained knowledge, we can now understand few more things from the command output.

```txt{linenos=false}
# COMMAND -->    usb-devices

T:  Bus=01 Lev=00 Prnt=00 Port=00 Cnt=00 Dev#=  1 Spd=480 MxCh=12
D:  Ver= 2.00 Cls=09(hub  ) Sub=00 Prot=01 MxPS=64 #Cfgs=  1
P:  Vendor=1d6b ProdID=0002 Rev=05.18
S:  Manufacturer=Linux 5.18.3-arch1-1 xhci-hcd
S:  Product=xHCI Host Controller
S:  SerialNumber=0000:00:14.0
C:  #Ifs= 1 Cfg#= 1 Atr=e0 MxPwr=0mA
I:  If#= 0 Alt= 0 #EPs= 1 Cls=09(hub  ) Sub=00 Prot=00 Driver=hub
E:  Ad=81(I) Atr=03(Int.) MxPS=   4 Ivl=256ms

T:  Bus=01 Lev=01 Prnt=01 Port=00 Cnt=01 Dev#=  5 Spd=12  MxCh= 0
D:  Ver= 2.00 Cls=00(>ifc ) Sub=00 Prot=00 MxPS= 8 #Cfgs=  1
P:  Vendor=046d ProdID=c534 Rev=29.01
S:  Manufacturer=Logitech
S:  Product=USB Receiver
C:  #Ifs= 2 Cfg#= 1 Atr=a0 MxPwr=98mA
I:  If#= 0 Alt= 0 #EPs= 1 Cls=03(HID  ) Sub=01 Prot=01 Driver=usbhid
E:  Ad=81(I) Atr=03(Int.) MxPS=   8 Ivl=8ms
I:  If#= 1 Alt= 0 #EPs= 1 Cls=03(HID  ) Sub=01 Prot=02 Driver=usbhid
E:  Ad=82(I) Atr=03(Int.) MxPS=  20 Ivl=2ms

T:  Bus=02 Lev=00 Prnt=00 Port=00 Cnt=00 Dev#=  1 Spd=5000 MxCh= 6
D:  Ver= 3.00 Cls=09(hub  ) Sub=00 Prot=03 MxPS= 9 #Cfgs=  1
P:  Vendor=1d6b ProdID=0003 Rev=05.18
S:  Manufacturer=Linux 5.18.3-arch1-1 xhci-hcd
S:  Product=xHCI Host Controller
S:  SerialNumber=0000:00:14.0
C:  #Ifs= 1 Cfg#= 1 Atr=e0 MxPwr=0mA
I:  If#= 0 Alt= 0 #EPs= 1 Cls=09(hub  ) Sub=00 Prot=00 Driver=hub
E:  Ad=81(I) Atr=03(Int.) MxPS=   4 Ivl=256ms
```

We already know that the output is grouped and what these characters at the most-left side mean. We now just need to map what rest of the things mean and how they relate to all what we have just learned.

So the first line in each group starts with `T`, which indicates the information in that line is related to topology of the device. `Bus` indicates what physical bus that device is connected to. `Lev` indicates the level of the node in the complete topology of that bus. Level 00 means it is the root hub. Next, level 01 will be any device connected to the main root hub (00) and all the devices connected to 01 hubs will be treated as level 02 devices and so on. `Spd` indicates the negotiated speed of that node. `MxCh` indicates how many devices can be connected to this device, and is 00 for anything except a hub.

Next line, starting with `D`, this shows the device information like `Ver` for USB version (mostly, 2 or 3 for now), `Cls` (class) of the device node. If this is marked as 00, then the interface should be read for the device class information. `Sub` indicates the sub-class of the node. `#Cfgs` indicate how many configurations this device has.

Next lines with `P` and `S` are usually the Vendor and Product IDs. Useful information if we want to write a driver for specific kinds of devices.

Then the remaining lines, starting with `C`, `I` and `E`, are the Configuration info, Interface info and the Endpoint info respectively. `#Ifs` tells about the total number of interfaces available for that device. `Cfg#` indicates the total number of available configurations for the device. `Atr` stores a hexadecimal value to indicate if the device is bus-powered(0x80), self-powered(0x40) or remote wake-up capable(0x20). `#Eps` indicates the endpoints for this alternate endpoint.


### USB driver - code walk-through

At this point, we roughly know what USB is comprised of and how it works inside linux kernel. Let's dig in the code.

```C
// SPDX-License-Identifier: GPL-2.0+

#include <linux/usb.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/hid.h>


static int hello_connect(struct usb_interface *interface, const struct usb_device_id *id)
{
	pr_alert("USB plugged in.\n");
	return 0;
}

static void hello_disconnect(struct usb_interface *interface)
{
	pr_alert("USB disconnected.\n");
}

static const struct usb_device_id id_table[] = {
	{
		USB_INTERFACE_INFO(
			USB_INTERFACE_CLASS_HID, 
	 		USB_INTERFACE_SUBCLASS_BOOT,
	  		USB_INTERFACE_PROTOCOL_KEYBOARD
		)
	},
    {
        USB_DEVICE(
            0x058f,  // Vendor Id
            0x6387   // Product Id
        )
    },
	{},  // End node - always null
};

MODULE_DEVICE_TABLE(usb, id_table);

static struct usb_driver driver = {
	.name = "ayedaemonUSB",
	.probe = hello_connect,
	.disconnect = hello_disconnect,
	.id_table = id_table,
};

static int __init hello_usb(void)
{
	pr_debug("Hello from ayedaemonUSB.\n");
	return usb_register(&driver);
}

static void __exit bye_usb(void)
{
	pr_debug("Bye from ayedaemonUSB.\n");
	usb_deregister(&driver);
}

module_init(hello_usb);
module_exit(bye_usb);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("ayedaemon");
MODULE_DESCRIPTION("Eudyptula task5");
```


You should be familiar with majority of the above code. This is how a simple linux module is written.

- First line is always the SPDX license. Read more [here](https://www.kernel.org/doc/html/latest/process/license-rules.html)
- Then some imports from other available header files.
- init (`hello_usb`) and exit (`bye_usb`) as entry and exit functions for the module.
- Macro calls for registering and unregistering both init and exit functions. Along with some module metadata macro calls.


Apart from this, we have some more code segment, which we are going to talk about in brief. At first, We need to create a `struct usb_driver` so that it can hold our driver information and the `id_table`. This `id_table` is very important for all the USB device drivers, as this list tells about the devices this driver can support. There are many macros that can help to define elements for this list. Each element is `struct  usb_device_id` [^13], and all the macros help to create this struct using only few values such as device class, product and vendor ids, etc.

[^13]: https://elixir.bootlin.com/linux/latest/source/include/linux/mod_devicetable.h#L127

```c
struct usb_device_id {
	/* which fields to match against? */
	__u16		match_flags;

	/* Used for product specific matches; range is inclusive */
	__u16		idVendor;
	__u16		idProduct;
	__u16		bcdDevice_lo;
	__u16		bcdDevice_hi;

	/* Used for device class matches */
	__u8		bDeviceClass;
	__u8		bDeviceSubClass;
	__u8		bDeviceProtocol;

	/* Used for interface class matches */
	__u8		bInterfaceClass;
	__u8		bInterfaceSubClass;
	__u8		bInterfaceProtocol;

	/* Used for vendor-specific interface matches */
	__u8		bInterfaceNumber;

	/* not matched against */
	kernel_ulong_t	driver_info
		__attribute__((aligned(sizeof(kernel_ulong_t))));
};

```


So when we load this module into a running kernel:-

- the `hello_usb` function will be executed and this in-turn will execute the `usb_register` function.
- `usb_register` function will register the device driver we created with `struct usb_driver`.
- This `usb_driver` will need few parameters like the name of the driver, functions to be called when this driver is loaded/unloaded by usb core. USB core will load this driver automatically, using hot-plug feature, when any usb device (which is supported by this driver) is plugged in.
- Along with other fields, `usb_driver` also contains a field that stores a list of all the devices this driver supports. This is used by USB core to know when to bind this driver with the usb device.
- `MODULE_DEVICE_TABLE` is the macro that exports the `id_table` to the usb core

#### Final steps

Now,all we need is a `Makefile` to make everything a bit automated and we are good to compile and load our USB module into the running kernel.

```makefile
CFLAGS_helloworld.o = -DDEBUG

obj-m += HelloWorld.o

KDIR ?= /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)

default:
	$(MAKE) -C $(KDIR) M=$(PWD) modules
clean: uninstall
	$(MAKE) -C $(KDIR) M=$(PWD) clean

install: default
	sudo insmod HelloWorld.ko
	lsmod | grep HelloWorld

uninstall:
	- lsmod | grep HelloWorld
	- sudo rmmod HelloWorld.ko

reload: uninstall clean default install
	@echo -e "\nDONE"
```


Just by doing `make install` we can compile and load the module into the kernel. There are multiple ways to monitor changes in kernel, for this case, I'll use `journalctl`, `udevadm` and `lsusb` commands to check the module messages, USB changes, and associated driver information for my device.

To begin with anything, we need to compile and install the module and check the messages from kernel.

Compile and load:-

```bash
make install
```

Check kernel logs:-

```txt{linenos=false}
### COMMAND:-  udevadm monitor --kernel

KERNEL[1736.016830] bind     /devices/pci0000:00/0000:00:14.0/usb1/1-4/1-4.3/1-4.3:1.0 (usb)
KERNEL[1736.016864] add      /bus/usb/drivers/ayedaemonUSBdriver (drivers)
KERNEL[1736.016888] add      /module/HelloWorld (module)


### COMMAND:-  journalctl --grep=usb -f

Jun 29 11:11:41 FatSaturn kernel: usbcore: registered new interface driver ayedaemonUSBdriver
Jun 29 11:11:41 FatSaturn kernel: USB driver registered this time


### COMMAND:-  lsmod | grep HelloWorld

HelloWorld             20480  0
```


The above logs helps us to identify that our module was loaded to the kernel successfully. Now time to insert the USB stick (pen-drive).

```txt{linenos=false}
### COMMAND:-  udevadm monitor --kernel

KERNEL[1981.372348] add      /devices/pci0000:00/0000:00:14.0/usb1/1-4/1-4.3 (usb)
KERNEL[1981.377254] add      /devices/pci0000:00/0000:00:14.0/usb1/1-4/1-4.3/1-4.3:1.0 (usb)
KERNEL[1981.378079] add      /devices/virtual/workqueue/scsi_tmf_3 (workqueue)
KERNEL[1981.379821] add      /devices/pci0000:00/0000:00:14.0/usb1/1-4/1-4.3/1-4.3:1.0/host3 (scsi)
KERNEL[1981.379930] add      /devices/pci0000:00/0000:00:14.0/usb1/1-4/1-4.3/1-4.3:1.0/host3/scsi_host/host3 (scsi_host)
KERNEL[1981.380070] bind     /devices/pci0000:00/0000:00:14.0/usb1/1-4/1-4.3/1-4.3:1.0 (usb)
KERNEL[1981.380229] bind     /devices/pci0000:00/0000:00:14.0/usb1/1-4/1-4.3 (usb)
KERNEL[1982.383440] add      /devices/pci0000:00/0000:00:14.0/usb1/1-4/1-4.3/1-4.3:1.0/host3/target3:0:0 (scsi)
KERNEL[1982.383621] add      /devices/pci0000:00/0000:00:14.0/usb1/1-4/1-4.3/1-4.3:1.0/host3/target3:0:0/3:0:0:0 (scsi)
KERNEL[1982.383811] add      /devices/pci0000:00/0000:00:14.0/usb1/1-4/1-4.3/1-4.3:1.0/host3/target3:0:0/3:0:0:0/scsi_device/3:0:0:0 (scsi_device)
KERNEL[1982.384405] add      /devices/pci0000:00/0000:00:14.0/usb1/1-4/1-4.3/1-4.3:1.0/host3/target3:0:0/3:0:0:0/scsi_disk/3:0:0:0 (scsi_disk)
KERNEL[1982.385312] add      /devices/pci0000:00/0000:00:14.0/usb1/1-4/1-4.3/1-4.3:1.0/host3/target3:0:0/3:0:0:0/bsg/3:0:0:0 (bsg)
KERNEL[1982.389641] add      /devices/virtual/bdi/8:32 (bdi)
KERNEL[1982.408429] add      /devices/pci0000:00/0000:00:14.0/usb1/1-4/1-4.3/1-4.3:1.0/host3/target3:0:0/3:0:0:0/block/sdc (block)
KERNEL[1982.408491] add      /devices/pci0000:00/0000:00:14.0/usb1/1-4/1-4.3/1-4.3:1.0/host3/target3:0:0/3:0:0:0/block/sdc/sdc1 (block)
KERNEL[1982.408532] add      /devices/pci0000:00/0000:00:14.0/usb1/1-4/1-4.3/1-4.3:1.0/host3/target3:0:0/3:0:0:0/block/sdc/sdc2 (block)
KERNEL[1982.410675] bind     /devices/pci0000:00/0000:00:14.0/usb1/1-4/1-4.3/1-4.3:1.0/host3/target3:0:0/3:0:0:0 (scsi)




### COMMAND:-  journalctl --grep=usb -f

Jun 29 11:15:46 FatSaturn kernel: usb 1-4.3: new high-speed USB device number 12 using xhci_hcd
Jun 29 11:15:47 FatSaturn kernel: usb 1-4.3: New USB device found, idVendor=058f, idProduct=6387, bcdDevice= 1.00
Jun 29 11:15:47 FatSaturn kernel: usb 1-4.3: New USB device strings: Mfr=1, Product=2, SerialNumber=3
Jun 29 11:15:47 FatSaturn kernel: usb 1-4.3: Product: Mass Storage
Jun 29 11:15:47 FatSaturn kernel: usb 1-4.3: Manufacturer: Generic
Jun 29 11:15:47 FatSaturn kernel: usb 1-4.3: SerialNumber: EFEC1147
Jun 29 11:15:47 FatSaturn kernel: usb-storage 1-4.3:1.0: USB Mass Storage device detected
Jun 29 11:15:47 FatSaturn kernel: scsi host3: usb-storage 1-4.3:1.0
Jun 29 11:15:47 FatSaturn mtp-probe[8926]: checking bus 1, device 12: "/sys/devices/pci0000:00/0000:00:14.0/usb1/1-4/1-4.3"
Jun 29 11:15:47 FatSaturn mtp-probe[8939]: checking bus 1, device 12: "/sys/devices/pci0000:00/0000:00:14.0/usb1/1-4/1-4.3"


### COMMAND:-  lsusb 

Port 3: Dev 13, If 0, Class=Mass Storage, Driver=usb-storage, 480M
```


Above output clearly shows that the pen-drive was detected but it was assigned to another driver instead of my custom driver. So we need to unbind my device with currently associated device driver and then bind it with our device driver. It is actually very easy to do in newer kernels (anything>=2.6.13-rc3). There are many resources for this topic on the Internet, but I found this one [short and precise article on lwn.net](https://lwn.net/Articles/143397/). TL-DR; I just had to invoke 2 commands from a terminal.


```bash
sudo /bin/bash -c 'echo 1-4.3:1.0 > /sys/bus/usb/drivers/usb-storage/unbind'
sudo /bin/bash -c 'echo 1-4.3:1.0l > /sys/bus/usb/drivers/ayedaemonUSBdriver/bind'
```

These are the files exposed by kernel to handle dynamic binding and unbinding from user-space. The number `1-4.3:1.0` the endpoint location specifier ... I mean this is a specifier that can be used to locate the actual endpoint for the USB that needs this driver.

If we break it down it'll be much more understandable.

```
##  1-4.3:1.0
1: root_hub
4: My USB hub (usb hub extension connected to laptop usb port)
3: USB device connected to that hub
1: Config number
0: Interface number
```

To summarize, the device naming scheme is somewhat like this -> `root_hub-hub_port.internal_port:config.interface`. So If I directly plug my pen-drive into my laptop USB port, it should give me something like `1-*:1.0`.. because I know I'm plugging it to bus 1 (It's my laptop, I know it), and then the `config.interface` part will be same as the old one. Interesting thing to note here is that since there is no external hub connected this time, so the `hub_port.internal_port` will just be `hub_port`. So I'm expecting only 1 value in that place. Hence, `1-*:1.0` and in the logs, I got `1-2:1.0`. 

![](https://media.giphy.com/media/Yq7ioTdDP5GzLSrqOw/giphy.gif#center)


In this era of laziness and automation, we will want to do some automation for binding/unbinding. What we would need is a program that keeps on listening on the device events and help us to run our commands when a particular event occurs.... `udev` can help us do that!!! All we have to do is write a simple rule and provide it to `udev` and the rest will be taken care of.

To create a udev rule, paste the below text in the following file - `/etc/udev/rules.d/99-custom-usb.rules`

```udev
ACTION=="bind", SUBSYSTEMS=="usb", \
OPTIONS="log_level=debug", \
ATTRS{idVendor}=="058f", ATTRS{idProduct}=="6387", \
RUN+="/bin/bash -c 'echo $kernel > /sys/bus/usb/drivers/usb-storage/unbind'", \
RUN+="/bin/bash -c 'echo $kernel > /sys/bus/usb/drivers/ayedaemonUSBdriver/bind'"
```

The above rule matches the USB Attributes and triggers the `RUN` commands accordingly. If you want this rule to be more generic, remove the `ATTRS{idVendor}` and `ATTRS{idProduct}`. [Read here](http://reactivated.net/writing_udev_rules.html) to learn more about writing udev rules.

If you change the C code and reload the module and remove the `ATTRS` selector from the udev rule, you can get a system where any `usb-storage` kind of devices will be automatically binded with `Driver=ayedaemonUSBdriver`.

```
Port 3: Dev 18, If 0, Class=Mass Storage, Driver=ayedaemonUSBdriver, 480M
ID 058f:6387 Alcor Micro Corp. Flash Drive

Port 1: Dev 17, If 0, Class=Mass Storage, Driver=ayedaemonUSBdriver, 480M
ID 0930:6545 Toshiba Corp. Kingston DataTraveler 102/2.0 / HEMA Flash Drive 2 GB / PNY Attache 4GB Stick
```

### Conclusion

Writing a Linux USB device driver is not a difficult task if one understands how USB subsystem works behind the abstractions. This article just touches the surface of USB drivers and there is still a lot more to look out for... like USB urbs. If you want to continue learning more about USB drivers, most common recommendation on internet is - [*Linux Device Drivers: Chapter 13. USB Drivers*](https://www.makelinux.net/ldd3/?u=chp-4-sect-2.shtml). This book is somewhat dated, but the content is still relevant.