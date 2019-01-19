---
title: "Deploy ConcourseCI with BOSH"
date: 2017-07-24T08:49:32+10:00
---

Deploying ConcourseCI with BOSH is easy, and there is a lot of information out there about it and how to make it work.

I'll go through how I deploy concourse locally for testing. I deploy to a director that is running locally in virtualbox

First thing we need to do is make sure that our director has some resources and the right stemcell.
<!--more-->
The fastest way to get these is to use bosh.io, so lets load the stemcell in first.

```bash
bosh -e vbox upload-stemcell https://bosh.io/d/stemcells/bosh-warden-boshlite-ubuntu-trusty-go_agent
```
Next we need to add the garden-runc release, concourse requires this release. Again, get it from bosh.io to make it easier.

```bash
bosh -e vbox upload-release https://bosh.io/d/github.com/cloudfoundry/garden-runc-release
```
Now we can either upload the release from bosh.io, or you can clone the repo and create-release then upload-release manually.
Using bosh.io is easier.

```bash
bosh -e vbox upload-release https://bosh.io/d/github.com/concourse/concourse
```

Sweet, we nearly have the requirements sorted. We need some configuration for our BOSH directory, if you have set up the bosh director using my other tutorial, you will likely be using this cloud-config.
```yaml
---
azs:
- name: z1
- name: z2
- name: z3

vm_types:
- name: default

disk_types:
- name: default
  disk_size: 1024

networks:
- name: default
  type: manual
  subnets:
  - azs: [z1, z2, z3]
    dns: [10.120.15.71]
    range: 10.244.0.0/24
    gateway: 10.244.0.1
    static: [10.244.0.2-10.244.0.200]
    reserved: []

compilation:
  workers: 5
  az: z1
  reuse_compilation_vms: true
  vm_type: default
  network: default
```
We need to tell our director to use it
```bash
bosh -e vbox update-cloud-config /path/to/cloud-config.yml
```
Now our director knows about our local virtualbox environment. We just need one more thing to deploy concourse. The deployment manifest.
```yaml
---
name: concourse

releases:
  - name: concourse
    version: latest
  - name: garden-runc
    version: latest

stemcells:
  - alias: ubuntu
    name: bosh-warden-boshlite-ubuntu-trusty-go_agent
    version: latest

instance_groups:
  - name: web
    instances: 1
    azs: [z1]
    stemcell: ubuntu
    vm_type: default
    networks:
      - name: default
        static_ips:
          - 10.244.0.2
    jobs:
      - release: concourse
        name: atc
        properties:
          postgresql_database: atc
          external_url: http://10.244.0.2:8080
          development_mode: true
          basic_auth_username: REPLACE_ME
          basic_auth_password: REPLACE_ME
      - release: concourse
        name: tsa
        properties: {}

  - name: db
    instances: 1
    azs: [z1]
    stemcell: ubuntu
    vm_type: default
    networks: [{name: default}]
    persistent_disk: 10240
    jobs:
      - release: concourse
        name: postgresql
        properties:
          databases:
          - name: atc
            role: atc
            password: dummy-postgres-password

  - name: worker
    instances: 1
    azs: [z1]
    stemcell: ubuntu
    vm_type: default
    networks: [{name: default}]
    jobs:
      - release: concourse
        name: groundcrew
        properties: {}
      - release: concourse
        name: baggageclaim
        properties: {}
      - release: garden-runc
        name: garden
        properties:
          garden:
            # cannot enforce quotas in bosh-lite
            disk_quota_enabled: false
            listen_network: tcp
            listen_address: 0.0.0.0:7777
            allow_host_access: true

update:
  canaries: 1
  max_in_flight: 3
  serial: false
  canary_watch_time: 1000-60000
  update_watch_time: 1000-60000
```
Replace the basic_auth_username and password, then save the config somewhere and we can tell our director to kick off the deployment.
```bash
bosh -e vbox -d concourse deploy /path/to/deployment-manifest.yml
```
Our director is off and running now, it can take some time to finish but once it's done you will be able to check out the concourse webUI at <http://10.244.0.2:8080>

You will need to download the fly cli tool, you can click here <http://10.244.0.2:8080/api/v1/cli?arch=amd64&platform=linux> for linux, it will download it from your local concourse directly.

Put it somewhere in your path and make it executable. Then run
```bash
fly -t tutorial login -c http://10.244.0.2:8080
```
Use the username and password you set up before in the deployment manifest.

Now you can use fly to manage your local concourse, use `fly -t tutorial <rest-of-fly-command>` to interact with concourse
