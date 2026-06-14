---
title:  "A template for using NixOS in a homelab"
date:   2026-06-14 12:00:00 -0400
draft: false
tags: ["homelab", "nixos"]
categories: ["homelab"]
---

# Introduction

Last year I bought a used Lenovo ThinkStation P510 from BestBuy to replace my cluster of RPIs that I had running in my home. With this change, I wanted a more reliable configuration for my services instead of the ad-hoc approach I took previously. One of the main challenges I encountered was trying to troubleshoot a broken service that I configured 6 months ago and remembering all the changes I had previously made. This is how I stumbled upon NixOS and from my experience it has the following pros & cons.

## Advantages of using NixOS

1. It allows for all my machines / services to be self-documenting
2. It's easy to propogate a change to every machine in my homelab
3. Common features can be configured through a simple copy-paste between files
4. In a recovery scenario, it should only take a few minutes to re-deploy all the services (though data needs to be backed up separately)

## Disadvantages of using NixOS

1. Takes time to understand how the NixOS functions, even for someone with a Linux background
2. Problems are still difficult to troubleshoot :P This relates to how packages are stored in the Nix Store.

# Template

To address point 1 of the disadvantages, I hope to provide a sensible framework to get people started with using NixOS in their homelab. The full template below is available on Github. This article will walk you through:

1. Creating your first VM with NixOS installed
2. Creating your first homelab service (in this case Nextcloud)
3. Configuring DNS within your homelab network
4. Configuring TLS for all of your homelab services

All of the advice I provide has been tested in my own homelab.

## Creating an initial VM

The first thing to do is to install NixOS. The [official documentation](https://nixos.org/manual/nixos/stable/#ch-installation) will guide you through the process. I decided to go with the [Minimal ISO image](https://nixos.org/download/#nixos-iso). I also avoided a *flake-based* configuration, which is more complicated and can always be configured later.

During the installation process, I modified the generated `configuration.nix` file to include the following lines. It's not the most secure approach but we will be improving it later.

```sh
services.openssh = {
  enable = true;
  settings.PermitRootLogin = "yes";
};
```

Once the installation process is complete, you should be able to boot directly into NixOS. For Proxmox, I also needed to disable *Secure Boot* to boot correctly. You should also be able to `ssh` into the machine as the root user, which means to can use `scp` to copy over a more complete `configuration.nix`. I **strongly** suggest you first copy the auto-generated `configuration.nix` from the NixOS machine to your current machine, make any modifications, then copy it back to the NixOS machine. Here's a sample of some of my configurations:

```sh
# Define a user account
users.users.mike = {
  isNormalUser = true;
  home = "/home/mike";
  description = "Default mike account";
  extraGroups = [ "wheel" ];
  shell = pkgs.bash;
  openssh.authorizedKeys.keys = [
    "<YourPublicSSHKey>"
  ];
};

# Add our private key to the root user so we can push NixOS config files
users.users.root = {
  openssh.authorizedKeys.keys = [
    "<YourPublicSSHKey>"
  ];
};

# Allow sudo without password
security.sudo.wheelNeedsPassword = false;

# List packages installed in system profile.
# You can use https://search.nixos.org/ to find more packages (and options).
environment.systemPackages = with pkgs; [
  vim
  wget
  curl
  htop
];

# Enable the OpenSSH daemon.
services.openssh = {
  enable = true;
  settings = {
    # Having root access is nice for scripts that will update the system, otherwise
    # copying system files can be challenging. IMO password auth is a bigger security concern.
    PasswordAuthentication = false;
    PermitRootLogin = "yes";
  };
};
```

Just modifying the configuration file isn't enough to change the system. For that you need to use the `nixos-rebuild` command. I use `nixos-rebuild test` to apply the changes and verify whether they work as expected. This command *does not* persist the changes after rebooting. Once I am satisfied with the changes, I use `nixos-rebuild switch` to apply and persist the changes. It is always good to reboot a machine to verify that the changes have persisted.

Once the system has been rebuilt, you should be able to SSH into your server without needing a password. This is essential for simple administration of multiple servers. If you are running NixOS on a VM within a hypervisor, the next logical step is create a VM template. This makes it easy to spin up new VMs for new services. And since they already have SSH-access enabled, they will be easy to modify.

## Service Configuration

Let's walk through installing Nextcloud. I start by reading the [Nextcloud NixOS wiki article](https://nixos.wiki/wiki/Nextcloud) to get a general idea of necessary configurations. I can also browse the more comprehensive [NixOS Configuration Options](https://nixos.org/manual/nixos/stable/options#opt-services.nextcloud.enable) for additional options. I then create a new file, `nextcloud.nix`, with all the configurations I want:

```sh
{ pkgs, ... }:

{
  services.nextcloud = {
    enable = true;

    # We will later change this to something like:
    #hostName = "nextcloud.mikes-homelab.com";
    hostName = "nextcloud.local";

    # Need to manually increment with every major upgrade
    package = pkgs.nextcloud32;

    # Let NixOS install and configure Redis caching automatically
    configureRedis = true;

    # Increase the maximum file upload size to avoid problems uploading videos
    maxUploadSize = "16G";

    config = {
      dbtype = "sqlite";
      adminpassFile = "/etc/nextcloud-admin-pass";
    };
  };

  # Permit access to the Nextcloud server through the firewall
  networking.firewall.allowedTCPPorts = [
    80
  ];
}
```

I generally install each service in its' own VM by cloning from the VM template we created in the previous step. Once that is done, I can push this file to VM using `scp` and store it as `/etc/nixos/nextcloud.nix`. I modify `/etc/nixos/configuration.nix` to include this new configuration file:

```
imports = [ ./hardware-configuration.nix ./nextcloud.nix ];
```

Nextcloud also an initial admin password, which we store at `/etc/nextcloud-admin-pass`.

Finally, we run `nixos-rebuild test` or `nixos-rebuild switch` for your changes to take effect. If all goes well, you should be able to access Nextcloud through a browser at `http://<VM-IP-address>`. The only issue is that Nextcloud maintains a list of trusted domains through which it can be accessed through. We can either add the VM IP and `nextcloud.local` (defined in `nextcloud.nix`) to a file like `/etc/hosts` or configure a DNS server to manage the domain names of all our VMs.

## DNS configuration

Configuring an internal DNS server provides a number of benefits including:

- a consistent way to contact your services, which helps with scripting
- ad-blocking at the DNS level for all your devices configured with your DNS server
- the ability to access services through domain names, which is needed for HTTPS connections (more on this later)

To achieve these tasks I used [Technitium DNS](https://technitium.com/dns/), though other options available. I initially tried installing Technitium through [NixOs configurations](https://nixos.org/manual/nixos/stable/options#opt-services.technitium-dns-server.enable), however I kept running into errors and realized that it was not (at least at the time) well supported. The next best approach, in my opinion, was to deploy Technitium through docker. I cloned a new VM to host all my docker images, and copied over a configuration containing:

```sh
# /etc/nixos/docker.nix
{ pkgs, ... }:

{
  environment.systemPackages = with pkgs; [
    docker-compose
  ];
  virtualisation.docker.enable = true;
  users.users.mike.extraGroups = [ "docker" ];

  # Assign a static IP address so your devices can reliable contact your DNS server
  networking = {
    # Assign the static IP address to the correct interface, in my case ens18
    interfaces.ens18 = {
      ipv4.addresses = [{
        address = "10.0.0.2"; # Pick an available IP address on your network
        prefixLength = 24;
      }];
    };

    # Need to define a default gateway on static IPs
    defaultGateway = {
      address = "10.0.0.1";
      interface = "ens18";
    };

    # Need to also define a set of nameservers on static IPs
    nameservers = [
      "8.8.8.8" # Google DNS server
    ];
  };

  networking.firewall = {
    allowedTCPPorts = [
      53    # DNS requests over TCP
      5380  # Technitium webUI over HTTP
    ];
    allowedUDPPorts = [
      53    # DNS requests over UDP
    ];
  };
}
```

After that configuration has been applied, you should be able to run `docker` as well as `docker-compose` commands. I used the `docker-compose.yml` from the [official Technitium repo](https://github.com/TechnitiumSoftware/DnsServer/blob/master/docker-compose.yml) to deploy my DNS server, and it looks like this:

```yaml
services:
  dns-server:
    container_name: technitium
    hostname: technitium
    image: technitium/dns-server:latest
    # For DHCP deployments, use "host" network mode and remove all the port mappings, including the ports array by commenting them
    # network_mode: "host"
    ports:
      - "5380:5380/tcp" #DNS web console (HTTP)
      - "53:53/tcp" #DNS service
      - "53:53/udp" #DNS service
    environment:
      - DNS_SERVER_DOMAIN=technitium.local #The primary domain name used by this DNS Server to identify itself.
      - DNS_SERVER_LOG_FOLDER_PATH=/var/log/technitium/dns #The folder path on the server where the log files should be saved.
    volumes:
      - config:/etc/dns
      - logs:/var/log/technitium/dns
    restart: unless-stopped
    sysctls:
      - net.ipv4.ip_local_port_range=1024 65000

volumes:
  config:
  logs:
```

After running `docker-compose up -d`, you should be able to access the Technitium webUI using `http://<VM-IP-address>:5380`. From there, you should be able to configure things like ad-blocking, and assigning domain names. I suggest purchasing a domain name from a registrar, then using that for all your internal services. This will make the TLS configuration in the next step very smooth, and it will also mean that you won't need to accept untrusted certificates on all your devices.

For the sake of this article, let's assume I was able to register the domain `mikes-homelab.com`. I would then create a (primary) zone on my technitium server, and add a new A record for `nextcloud.mike-homelab.com`, pointing to the nextcloud IP. I would also need to change the hostName variable in the `nextcloud.nix` configuration to match this domain.

Finally, I update desktop to use my Technitium server for DNS resolution. I can test that everything works using the `dig` command.

```sh
❯ dig nextcloud.mikes-homelab.com

; <<>> DiG 9.20.23 <<>> nextcloud.mikes-homelab.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 42744
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;nextcloud.mikes-homelab.com.   IN      A

;; ANSWER SECTION:
nextcloud.mikes-homelab.com. 3600 IN    A       10.0.0.199

;; Query time: 16 msec
;; SERVER: 10.0.0.105#53(10.0.0.105) (UDP)
;; WHEN: Sun Jun 14 13:03:40 EDT 2026
;; MSG SIZE  rcvd: 72
```

I am now able to access Nextcloud through the browser at `http://nextcloud.mikes-homelab.com`!

## TLS configuration

TLS configuration can be confusing at first, but once it's setup NixOS makes it easy to configure for every subsequent VM.

Let's start by going over some key concepts. First, a TLS certificate contains a list of one more domains that it applies to. This list of domains is verified by a certificate authority to ensure the user that is requesting a certificate indeed owns those domains. As an example. the process of obtaining a TLS certificate might look like this:

1. I purchase the domain `mikes-homelab.com` from a *domain registrar* like GoDaddy
2. I request a TLS certificate for `nextcloud.mikes-homelab.com` from a Certificate Authority (CA) like Let's Encrypt
3. The CA asks that I prove that I indeed own `nextcloud.mikes-homelab.com` through a *challenge* (more on this later)
4. Once the CA verifies that I own the domain, it issues a TLS certificate

There are two most common challenges are **HTTP-01** and **DNS-01** challenges. The first asks the user to create a file at a well-defined path on the server. For example, Let's Encrypt may ask that we create the file `http://nextcloud.mikes-homelab.com/.well-known/acme-challenge/XYZ`, then checks whether the file exists. If we are able to create this file, then it is implied that we own the domain name.

Since most homelab services are not accessible from the Internet, the better approach for us is a **DNS-01** challenge. Instead of creating a file on a server, this type of challenge asks us to create a specific TXT record in our domain. This can be done for free and automatically using a *DNS provider* like Cloudflare, which provides an API that NixOS supports.

The first step is to create a Cloudflare account and [onboard your domain](https://developers.cloudflare.com/fundamentals/manage-domains/add-site/). Then I can create an [API Token](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/). The token value will be stored on every VM so that it can automatically create and renew certificates. Then I added the following stanza to my `configuration.nix` file, as almost every VM has service that operates over HTTP.

```sh
{
  ...

  security.acme.acceptTerms = true;
  security.acme.defaults = {
    email = "myemail@gmail.com";
    dnsProvider = "cloudflare";
    environmentFile = "/path/to/cloudflare-api-token";
  };

  ...
}
```

After that, NixOS makes it very simple to setup an nginx reverse proxy, and configure it with TLS certificates. For example, we could add this snippet to our `nextcloud.nix` file:

```sh
{
  ...

  networking.firewall.allowedTCPPorts = [
    80
    443
  ];

  services.nginx = {
    enable = true;
    virtualHosts = {
      "nextcloud.mikes-homelab.com" = {
        acmeRoot = null;
        enableACME = true;
        forceSSL = true;
      };
    };
  };

  ...
}
```

And if our service runs on a non-default port, like the Technitium web UI, we can just add:

```sh
{
  ...


  networking.firewall = {
    allowedTCPPorts = [
      53   # DNS requests over TCP
      80   # Technitium webUI through nginx over HTTP
      443  # Technitium webUI through nginx over HTTPS
    ];
    allowedUDPPorts = [
      53    # DNS requests over UDP
    ];
  };

  services.nginx = {
    enable = true;
    virtualHosts = {
      "dns.mikes-homelab.com" = {
        acmeRoot = null;
        enableACME = true;
        forceSSL = true;
        locations."/" = {
          proxyPass = "http://127.0.0.1:5380";
        };
      };
    };
  };

  ...
}
```

Once all of those changes have been made, you should be able to access Nextcloud over HTTPS!

# Conclusion

Using NixOS for a homelab provides a number of benefits, including self-documentation and the ability to quickly copy and propagate features to multiple VMs. At the same time, it can be challenging to work with and troubleshoot issues. Whether or not NixOS is right for you depends on your particular use case. For myself, I (mostly) enjoyed learning about the nuances of NixOS, and I am very happy with how easy to manage and add new services to my homelab.

I hope you found this article useful in your journey of adopting NixOS!
