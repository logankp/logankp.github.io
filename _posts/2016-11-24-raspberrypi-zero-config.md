---
layout: single
categories: [linux, technology, raspberrypi]
---

I've recently acquired a [Raspberry Pi Zero](https://www.raspberrypi.org/products/pi-zero/) and wanted an easy way to set up networking through USB. For initial configuration, see [this Gist](https://gist.github.com/gbaman/975e2db164b3ca2b51ae11e45e8fd40a). After getting this set up, it still didn't "just work" on my machine. The following configuration was required:

1. Install avahi-daemon and avahi-autoipd
    1. Ideally after this step you should be able to ping raspberrypi.local, but this didn't work for me.
        1. Avahi-autoipd didn't work because the interface name (enp0s3u1u3 or something like that) was too long to assign an alias: https://bugs.centos.org/view.php?id=11293
        2. To fix this, I had to use Udev to assign a shorter name to the interface (rpi0). This can be achieved by creating a /etc/udev/rules.d/99-raspberrypi.rules file:
       `SUBSYSTEM=="net", SUBSYSTEMS=="usb", ATTRS{idProduct}=="a4a2", ATTRS{idVendor}=="0525", NAME="rpi0"`
        3. After creating this file, restart avahi-autoipd and you should get a 169.254.\*.\* IP address on the rpi0:avahi interface, which should allow you to SSH in to your device.
2. Allow the pi to access the internet
    1. This requires enabling ip forwarding on your computer: `sysctl -w net.ipv4.ip_forward=1`
    2. Then set up an iptables rule to enable masquerading: `/sbin/iptables -t nat -A POSTROUTING -o <external_inetrace_name> -j MASQUERADE`
3. You now need to set up the correct route and DNS settings on the Pi:
    1. `route add default gw <IP address of rpi0:avahi interface>`
    2. `echo <DNS server IP address> > /etc/resolv.conf`
4. You should now be able to access the internet through your Pi Zero!

If you're like me, you probably want these rules to take effect when the Pi is plugged in and go away when it's unplugged. Thankfully this can be achieved with Udev and systemd:

1. Add the following line to your /etc/udev/rules.d/99-raspberrypi.rules: `SUBSYSTEM=="net", SUBSYSTEMS=="usb", NAME=="rpi0", ENV{SYSTEMD_WANTS}="avahi-autoipd@$name.service rpi-iptables.service"`
    1. This will trigger two services (which we will create in the next steps) to be fired when the Pi is plugged in.
2. Create two systemd services in /etc/systemd/system/

    avahi-autoipd@.service

        
        [Unit]
        Description=Assigns an ipv4LL address on the given interface

        [Service]
        ExecStart=/usr/sbin/avahi-autoipd %I
        ExecStop=/usr/sbin/avahi-autoipd -k %I
        
    rpi-iptables.service

        [Unit]
        Description=Set up iptables NAT rules when Raspberry Pi 0 is plugged in
        BindsTo=sys-subsystem-net-devices-rpi0.device

        [Service]
        ExecStart=/sbin/iptables -t nat -A POSTROUTING -o wlo1 -j MASQUERADE
        ExecStop=/sbin/iptables -t nat -F
        Type=oneshot
        RemainAfterExit=yes

3. Plug in your Pi Zero. Everything should be set up for you now (except step 3).

I have a [github repo](https://github.com/logankp/rpi-zero-config) with these services and an install script. Feel free to use it!
