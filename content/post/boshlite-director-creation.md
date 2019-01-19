---
title: "BOSHLite Director Creation"
date: 2017-07-23T08:49:32+10:00
---

In this tutorial, we are going to start up a local BOSHLite director in VirtualBox.

<!--more-->
You'll need to make sure you have VirtualBox installed before proceeding, you can get it [here](https://www.virtualbox.org/wiki/Downloads). Make sure you get the extension pack too for good measure.

Once you've got that sorted, you will need to clone the bosh-deployment repository from CloudFoundry [here](https://github.com/cloudfoundry/bosh-deployment).

Clone it somewhere you will remember. If you want to keep it all close to each other, for this example you could create a `~/bosh` directory and clone it into there. 

We will work in this directory for the remainder of the tutorial
```bash
mkdir -p ~/bosh; cd bosh
git clone https://github.com/cloudfoundry/bosh-deployment
```

We want to create a directory to store our virtualbox specific settings, so create a vbox directory inside `~/bosh`
```bash
mkdir -p ~/bosh/vbox
```

Right, next we need the actual BOSH CLIv2 tool, grab that from [here](https://bosh.io/docs/cli-v2.html#install) and follow the installation steps 1 and [2](https://bosh.io/docs/cli-env-deps.html) for your operating system.

If you run `bosh -v` and have version 2.x then we are good to go, if not check the installation for the CLIv2 again and make sure you didn't miss something.

We need to create a file `~/bosh/vbox/vbox.yml` with some information for our director to use when its built, mainly network config.
```yaml
---
director_name: "Bosh Lite Director"
internal_ip: 192.168.50.6
internal_gw: 192.168.50.1
internal_cidr: 192.168.50.0/24
outbound_network_name: NatNetwork
```

Now we have our local env set up, we want to create our BOSH director. Change into `~/bosh` and run the following
```bash
bosh create-env bosh-deployment/bosh.yml \
  --state vbox/state.json \
  -o bosh-deployment/virtualbox/cpi.yml \
  -o bosh-deployment/virtualbox/outbound-network.yml \
  -o bosh-deployment/bosh-lite.yml \
  -o bosh-deployment/bosh-lite-runc.yml \
  -o bosh-deployment/jumpbox-user.yml \
  --vars-store=vbox/creds.yml \
  --vars-file=vbox/vbox.yml
```

Breaking down the command, we can see `create-env` this is how BOSH bootstraps a director. It stores state information about the director into the `--state` file.

The `-o` ticks specify operations files, these override or add settings that are in `bosh-deployment/bosh.yml` to suit our VirtualBox environment. You shouldn't need to adjust any of these files.
If you do want to adjust them, you should use an operations file to do it it and keep it inside the `~/bosh/vbox/` directory so you aren't modifying the bosh-deployment repository.

The `--vars-store=vbox/creds.yml` file is created automatically, and is populated with automatically generated passwords and certificates for use with our director.

Once you run this command, it will start the process to create your BOSHLite director. Once it has completed, you can create an environment alias so you can access the director easily.
```bash
bosh -e 192.168.50.6 alias-env vbox --ca-cert <(bosh int vbox/creds.yml --path /director_ssl/ca)
```

This will alias 192.168.50.6 to vbox, so you can then use `bosh -e vbox` to access this director.

Now you can log in to the director and start using it, grab the password from the creds file that was automatically generated, then log in using the username admin
```bash
bosh int vbox/creds.yml --path /admin_password
bosh -e vbox login
```

Once you're logged in you should be able to check the environment information using `bosh -e vbox env` and it should output something similar to the following
```bash
Using environment '192.168.50.6' as client 'admin'

Name      Bosh Lite Director
UUID      ac582faf-081f-481b-8b33-c458b679f155
Version   262.3.0 (00000000)
CPI       warden_cpi
Features  compiled_package_cache: disabled
          config_server: disabled
          dns: disabled
          snapshots: disabled
User      admin

Succeeded
```

That's it! Done. Now you're ready to start mucking around with BOSH locally, check out my other tutorials as they come.
