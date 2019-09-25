# Docker: Resolving DNS issues on OVH

## The Issue

One of the many lower-priority issues discussed in the annals of Docker's Github 
issue pages has to do with an apparent issue of being 
[unable to resolve docker.io repositories from within Ubuntu on OVH servers](
https://github.com/dotcloud/docker/issues/1470#issuecomment-45936411).

## What's causing it?

I'm not really sure anymore. I worked through the issue about a month ago, but 
can tell you this:

  1. It's not necessarily due to the custom Kernel that OVH loads into their 
      servers. Well, at least if they actually do give you the distribution 
      kernel when you deploy a custom build and uncheck the option to use their 
      kernel. 

      However, you should never use the OVH custom kernel if you plan on using 
      linux containers. Just install lxc and run

      ```bash
            lxc-checkconfig
      ```

      to see a list of incompatibilities [0].

      You can switch out your kernel or ensure that you deselect the option to use a 
      custom kernel when you're going through the set-up process for your host 
      machine.
  2. I had issues with containers running on a host which was running Ubuntu
       14.04 being unable to resolve any DNS.<
  3. If you're not on OVH but landed here because you're having issues with 
      your firewall and containers, you should note that adding IPTables rules 
      after your docker daemon has started is a bad idea. You can see why just
       by glancing at the 
       [Network Driver](
       https://github.com/dotcloud/docker/blob/master/daemon/networkdriver/bridge/driver.go#L176-L239)

## Fix it fix it fix it fix it fix it fix it

There's no need to uninstall bind9 or resolvconf. Simply use Google's public 
DNS servers instead by adding them to your box's network config.

Don't add the definitions to /etc/resolv.conf, as they'll be removed on reboot. 
Add them to /etc/network/interfaces on your **host** like so:

```bash
  # /etc/network/interfaces

  auto lo
  iface lo inet loopback

  auto eth0
  iface eth0 inet static
          address xxx.xxx.xxx.xxx
          netmask xxx.xxx.xxx.xxx
          network xxx.xxx.xxx.xxx
          broadcast xxx.xxx.xxx.xxx
          gateway xxx.xxx.xxx.xxx
          dns-nameservers 127.0.0.1 8.8.8.8 8.8.4.4

  iface eth0 inet6 static

  ...
```

Now, if you also have DNS issues within your containers, it's easy to tell the 
Docker daemon to provide Google's public DNS servers to your containers as well. 
There's more than one way to set these options, but, let's just be clear that 
editing the upstart conf file found in /etc/init/docker.io.conf is not the most 
ideal place to make these types of changes.

Let's make these changes in the idiomatic place: your host's 
/etc/default/docker.io file:

```bash
  # /etc/default/docker.io
  DOCKER_OPTS="-H unix:///var/run/docker.sock --dns 8.8.8.8 --dns 8.8.4.4"
```

Then, run these commands at the terminal to ensure the changes are set, 
replacing eth0 with your network interface:

```bash
  ifdown eth0 && ifup eth0 && service docker.io restart
```

La voila, you're all set. If you had been screwing around with your network 
prior to reading this article, it might be a good idea to reboot before and 
after these changes.

Hope that helps some poor soul - sorry for the quick and dirty post; send me a 
note if you want clarification on any piece!

Mo

[0] [http://lxc.sourceforge.net/man/lxc.html](http://lxc.sourceforge.net/man/lxc.html)
