# Introduction

ASiDe (A Simple Deployment pipeline) is a Continuous Integration and Deployment Pipeline. It is meant to quickly, reliably and transparantly roll out updates with fixes and new features of projects. ASiDe aims to be technology agnostic and only depends on git, inotify and bash, and optionally on a mail server.

When the applied build script contains the needed test, build and deployment steps, all that is needed is a push to a master branch to release the new version of your project. Logs can be followed or emailed to the user afterwards.

ASiDe consists of the following parts (replace projectname with the name of your project):

* A systemd-unit for turning the build trigger on or off: `/etc/systemd/system/aside.service`
* A monitoring script using `inotify(7)` to monitor which triggers appear in `/home/builduser/trigger/`: `/home/builduser/bin/monitor`
* A post-update hook in each git repository: `/srv/git/projectname.git/hooks/post-update`
* Build scripts which are triggered by the monitor script: `/var/home/builduser/projectname/bin/build`

# Flow

Deploying is done by a dedicated user `builduser`. Its home `/var/home/builduser/` looks as follows:

```
 /var/home/
 	 builduser/
 		  bin/monitor
 		  trigger/
 			   projectname
 		  projectname/
 	 		  bin/build
 	 		  resources/
 		 		   test/
 		 	  work/
```

The monitoring script uses the name of the trigger to select the correct project. Each project has a dedicated build script and resources used for testing when necessary. This way the development environment doesn't need to have those and they don't need to be committed to git, which could be insecure.

When the systemd-unit is enabled, it is started by booting the build server. Then the flow is as follows:

1. A developer pushes a change to `origin/master`.
2. The post-update hook checkt that the branch is `master`.
3. The post-update hook clones the repository to `/var/home/builduser/projectname/work/`.
4. The post-update hook creates an empty file with the name of the repository in `/var/home/builduser/trigger/` and deletes it immediately.
5. The monitoring script notices the trigger and starts the build script in the repository with the name of the trigger.
6. Afterwards is emails the build script's output to the developer.

The example build script deploys a Java Maven project. It runs the unit tests and copies the test resources to the project to run the integration tests. It then packages the project and copies the artifcact to an installation directory, to which builduser must have write rights. At last, it emails the logs to the developer and empties the working directory.

# Starting and stopping

Start, stop, enable and disable as you would any systemd unit. The unit is called `aside`.

# Installation

1. Create a system user `builduser` and a system group `deploy` with home directory `/var/home/builduser`. Use a free group number under 1000, to indicate it is a system group. The example uses 900.

```
# groupadd -r -g 900 deploy`
# useradd -r -b /var/home -m -N -g deploy -s /bin/bash builduser
```

2. Create a directory structure under `/var/home/builduser`.

```
# mkdir -p /var/home/builduser/{bin,trigger,project1/{bin,log,resources/{test,prod},work}}
```

3. Put the scripts at the right location and adapt them to your needs. See the diagram above.

4. Enable the triggers.

```
# systemd enable aside
```
