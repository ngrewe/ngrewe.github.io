---
layout: post
title: "Integrate a CoreOS host into an OpenVPN-based VPN"
comments: true
category: "Howto"
tags: [ "CoreOS", "Docker", "OpenVPN", "host networking" ]
---

I run a OpenVPN based private network and lately wanted to integrate a CoreOS
host with it so that etcd instances behind the VPN could participate in the
cluster. Turns out that this is a bit of hassle because you're supposed to run
everything in containers in CoreOS, except etcd and a few other choice components,
so the virtual network needs to be available in the host. Forunately, it can
easily be done (part of the configuration is loosely based on
[dperson/openvpn-client](https://github.com/dperson/openvpn-client)).

### Create a Docker image with OpenVPN

I build a new docker image  for this:
It just installs the openvpn package, cleans up and provides an attachment point
for the  volume where we will store the configuration:

    FROM debian:jessie
    MAINTAINER foo@example.com
    RUN export DEBIAN_FRONTEND='noninteractive' && \
        apt-get update -qq && \
        apt-get install -qqy --no-install-recommends openvpn \
                    $(apt-get -s dist-upgrade|awk '/^Inst.*ecurity/ {print $2}') &&\
                    apt-get clean && \
                    rm -rf /var/lib/apt/lists/* /tmp/*

    VOLUME ["/vpn"]
    ENTRYPOINT ["openvpn"]
    CMD [ "--config", "/vpn/vpn.conf" ]

### Add a systemd unit

Once the image is in place, you can put your VPN configuration in a folder of
your choice on the host (or stuff it into a data container, if you like). There
shouldn't be any changes required. Now create a systemd unit to run the openvpn
container, e.g. `/etc/systemd/system/openvpn.service`:

{% highlight ini %}
[Unit]
Description=OpenVPN
After=docker.service
Requires=docker.service

[Service]
TimeoutStartSec=0
Environment="OVPN_VOL=/etc/openvpn"
Environment="OVPN_IMAGE=1e3548f41a61"
Environment="OVPN_CONTAINER=openvpn"
ExecStartPre=-/usr/bin/docker stop ${OVPN_CONTAINER}
ExecStartPre=-/usr/bin/docker rm ${OVPN_CONTAINER}
ExecStart=/usr/bin/docker run --cap-add=NET_ADMIN \
  --device /dev/net/tun --name ${OVPN_CONTAINER} -v ${OVPN_VOL}/:/vpn \
  --net=host 1e3548f41a61 --config /vpn/vpn.conf
ExecStop=/usr/bin/docker stop ${OVPN_CONTAINER}

[Install]
WantedBy=multi-user.target
{% endhighlight %}

Obviously, you should replace the OVPN_* environment variables with values that
make sense to you:

* OVPN_VOL: The volume (on the host) where you store the configuration. If you
  are using a data container for this, use the `--volumes-from` switch instead
  of  `-v`
* OVPN_IMAGE: The name or ID of the image you've built.
* OVPN_CONTAINER: The name you want the running container to have


The magic here is
`-cap-add=NET_ADMIN --device /dev/net/tun --net=host`, which grants the container
enough privileges to set up the tun devices, and makes it visible in the host's
networking stack. Enable and start as follows:

{% highlight bash %}
sudo systemctl enable openvpn.service
sudo systemctl start openvpn.service
{% endhighlight %}

### Configure networking

This may be optional if you are using a plain tun device with server-assigned
IP configuration, but I actually have a setup with a tap device, so I created
a new systemd-networkd config file `/etc/systemd/network/openvpn.network`:

{% highlight ini %}
[Match]
Name=tap0

[Network]
Address=172.16.XXX.YYY/20
DNS=172.16.YYY.ZZZ
Domains=~example.com
{% endhighlight %}

This will automatically configure the network interface when the VPN connection
is established. The last line is actually fun, the ‘~’ does not set up a default
search domain, but instead tells the resolver to  prefer the configured
nameserver when resolving example.com domains. Pretty handy when dealing with
networks that have a bit of perspective to them (i.e. domain names resolve to
different IPs from the inside and the outside). Reload configuration to apply:

{% highlight bash %}
sudo systemctl restart systemd-networkd
sudo systemctl restart systemd-resolved
{% endhighlight %}
----
_Voila!_ Your CoreOS host is connected to the VPN.
