---
title: "Concourse Slackbot"
date: 2019-02-24T10:24:34+11:00
#draft: true
---

A while ago I wrote a slack bot where I worked that allowed the dev team to interact with our ConcourseCI instance set up in an isolated VPC in AWS. This bot did other things too, like allowing them to check AWS VGW VPN status, Pingdom status, and GitHub release versions. All of which were useful at the time.

Our Concourse also had notify on job failure and success, but this is only so helpful. Being able to trigger a job from slack made it so much better.

<!--more-->
The primary function of the bot was to allow the developers to trigger build jobs, or promote releases without having to have them log in to the VPN. And only using their knowledge of the CI/CD Pipelines to judge when and when not to get Concourse to start a job.

### Trimmed down
I did a re-write of the code, stripped out all of the AWS/Pingdom/GitHub functionality and brought it to the basics of just interacting with ConcourseCI.

You can find the code [here](https://github.com/shreddedbacon/concourse-slackbot)

I even created a BOSH release for the bot, so you can deploy it into the same environment that your Concourse is running.

Get the BOSH release [here](https://github.com/shreddedbacon/concourse-slackbot-boshrelease/releases/latest)

I tried to make the configuration simple. You can restrict individual commands from being used to a user level (perfect for Free slack versions), you can have it return the build output, or ignore it.

At the moment, it only supports triggering builds, and waiting for them to complete. But I plan to add the ability to perform resource checking, and pausing and unpausing of jobs.

### Configuration
In the `config.json` is where you configure the inital bits, like the name the bot responds to, the slack token, Concourse URL and username and password, and the commands that it accepts.

Configuring commands that the bot can run is pretty easy. Just add to the `commands` block in the config.json file.
```json
"commands":[
   {
      "command":"run pipeline-a job-a",
      "type":"concourse",
      "accept_response":"Got it, I'll get Concourse to start that for you now, it can take some time to do, but I'll let you know when it is done.",
      "help":"Run Job-B in Pipeline-A in Concourse; user must be in privileged users list for this bot to run",
      "options":{
         "team":"main",
         "pipeline":"pipeline-a",
         "job":"job-a",
         "skipoutput":false,
         "privileged":true
      },
      "privileged_users":[
         "USLACKBOT"
      ]
   },
   {
      "command":"run pipeline-a job-b",
      "type":"concourse",
      "accept_response":"Got it, I'll get Concourse to start that for you now, it can take some time to do, but I'll let you know when it is done.",
      "help":"Run Job-B in Pipeline-A in Concourse",
      "options":{
         "team":"main",
         "pipeline":"pipeline-a",
         "job":"job-b",
         "skipoutput":true,
         "privileged":false
      }
   }
]
```
The `options` block is where you configure the main bits of how it interacts with concourse. The `command` section is where you write the command you want to listen for, it doesn't have to match the pipeline and job name, just something you want to use. Use the `help` section to write a brief description that is show when you `@bot help`.

Using the `privileged_users` block, you can specify the user IDs of people that can run that command. I would like to expand this to groups at some point, but I have no way to test this properly on free slack plans.

Once the bot is running, you can run `@concoursebot help` and it will list all the commands you can use.

![SlackbotHelp](/img/slackbot-01.png)

Run a job, and wait for the result.

![SlackbotJobRun](/img/slackbot-02.png)

Or skip the output

![SlackbotSkipoutput](/img/slackbot-03.png)

Enjoy running concourse jobs from slack! Feel free to submit any issues or PRs if you see something you might want.
