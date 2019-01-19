---
title: "MACH - Fly Faster"
date: 2017-08-11T08:49:32+10:00
tag: ["fly","concourse","bosh", "mach"]
---

I got sick of typing out long commands for fly when doing development work with my concourse pipelines, so I wrote a wrapper for it called MACH.

<!--more-->

### The issue?
Fly doesn't appear to support any environment variables, and working with concourse and the way I do my pipelines I found myself often typing out similar commands in fly, but just changing the pipeline or job names.

I thought about it and noticed that normally I just change one option each time. I started messing around with aliases, but in the end just wrote the wrapper.

I broke down what I was repeating and what I was changing with my usage of fly and found that often my pipeline file is under `ci/pipeline.yml` and any custom variables are in `ci/vars.yml` and I was always targetting the one concourse instance `concourse` and a pipeline `pipeline`. A normal fly session could be something like this:
```bash
fly -t concourse set-pipeline -p pipeline -c ci/pipeline.yml -l ci/vars.yml
fly -t concourse unpause-pipeline -p pipeline
fly -t concourse trigger-job -j pipeline/jobname
fly -t concourse watch -j pipeline/jobname
```

If I changed to another directory containing a similar pipeline setup, the only thing that would change would be the pipeline name typically.

### How could this be faster?
```bash
# set and unpause the pipleine in one command
mach spup
# set and unpause with two commands
mach sp
mach up
# trigger and watch a job in one command
mach tjw
# trigger and watch in two commands
mach tj
mach w
```

Since I usually work in the directory that my pipeline lives, and majority of my pipelines are set out the same way, I created a `.mach` file with some basic variables in it that I place in each pipelines root directory.
```bash
FLYTARGET=concourse
PIPEFILE=ci/pipeline.yml
VARSFILE=ci/vars.yml
PIPELINE=pipeline
```

When executing mach, it checks in the working dir for this file, and sources the variables from it.

Any `job` based fly commands will collect a list of the jobs from the pipeline file and present them as user selectable, but you can specify just the job name using `mach tj -j jobname` and not need to use the pipeline name as it is sourced from the `.mach` file
```bash
user@host: ~/nginx-boshrelease$ mach tj
    __  ______   ________  __
   /  |/  /   | / ____/ / / /
  / /|_/ / /| |/ /   / /_/ /
 / /  / / ___ / /___/ __  /
/_/  /_/_/  |_\____/_/ /_/
=============================
Using pipeline: nginx-boshrelease
Pipeline file: ci/pipeline.yml
Vars file: ci/vars.yml

Trigger job in nginx-boshrelease

Select which job, or exit-mach:
1) test-release
2) bump-rc
3) bump-minor
4) bump-major
5) promote-release
6) Q) exit-mach
```

MACH doesn't do the full command set of fly, just the commands I found to use the most and wanted to speed them up.

See the github page for MACH to see what commands it supports.

### Where to get it?
Get MACH [here - https://github.com/shreddedbacon/mach](https://github.com/shreddedbacon/mach)
