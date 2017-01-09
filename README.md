# bosh-workshop
Bosh Workshop for Creating a Spring Music Release

# Pre requisites

1. bosh-cli
```
gem install bosh_cli --no-ri --no-rdoc
```

Install instructions: https://bosh.io/docs/bosh-cli.html

# 1. Initialize the bosh release directory
```
cd ~/workspace
bosh init release spring-music-release-<name> --git
```

This creates a release dir, with the following folder structure:
```
.
├── blobs
├── config
│   └── blobs.yml
├── jobs
├── packages
└── src
```
## Description of components: 
**blobs**: Normally contains binary files and third party dependencies of your release. These are usually things you don't want to directly commit in your repo. The spring-music release will have openjdk.tar.gz as a blob.

**config**: Configuration for an external file store where bosh can find your blobs. 

**jobs** Scripts that specify some work that bosh will execute on the VM. For spring-music, this will be running the spring-music.jar to start up the server.

**packages**: Scripts that specify how to copy content from your src and blobs into the VM. Scripts can do different things, like untar or compile source code before putting it on the VM.

**src**: Normally source code that the release relies on. Usually a submodule of some other repo.

# 2. Add your src dependencies
```
cd src
git submodule add https://github.com/ebabani/spring-music
cd ..
```
Add the spring-music project as a submodule.

# 3. Add openjdk as a blob
```
cd blobs
mkdir openjdk
cd openjdk
wget https://java-buildpack.cloudfoundry.org/openjdk-jdk/lucid/x86_64/openjdk-1.8.0_25.tar.gz
cd ../..
```

# 4. Create the openjdk package
A package is a collection of files that can be included as a dependency by other packages or jobs.
For openjdk, we want the package to be the untared openjdk archive we have in our blobs. 
Any other package or job including openjdk will see the content under /var/vcap/packages/jdk when they're running.

```
bosh generate package openjdk
```
This will create the following
```
packages/openjdk/
├── packaging
├── pre_packaging
└── spec
```
**spec**: yml file with information about the package we're creating. Contains the following:
```
---
name: openjdk

dependencies:

files:
```
- **name**: Package name
- **dependencies**: Packages can depend on other packages. openjdk won't depend on anything.
- **files**: Files that this package depends on. When looking for files, bosh will first look in src, then blobs. Openjdk will depend on the tar.gz file we added to the blobs. 

Update the spec file:
```packages/openjdk/spec
---
name: openjdk

dependencies: []

files:
- openjdk/**/*
```

**packaging**: Script to run to do the packaging.
When doing packaging, bosh will create a compilation VM and run your packaging script there.

The files your package depends on will be copied to $BOSH_COMPILE_TARGET directory on the vm
Any other packages you depend on will be found under /var/vcap/packages/{package name}

BOSH will then run your package script. 

After the script is done, BOSH will look in the $BOSH_INSTALL_TARGET directory on the compilation VM. That folder should contain the final contents of your package (eg. untared files, compiled software etc.). 

When a future package or job depends on your package, the contents of the $BOSH_INSTALL_TARGET will be present at /var/vcap/packages/{your package name}

For the openjdk package, we want want to untar the file, and put the contents in the $BOSH_INSTALL_TARGET directory
```
# abort script on any command that exits with a non zero value
set -ex

cd ${BOSH_INSTALL_TARGET}
tar xzvf ${BOSH_COMPILE_TARGET}/openjdk/*.tar.gz
```

**pre_packaging**
Similar to the packaging script, but the command will be run locally instead of a compilation VM. Not used much in practice.

After this step, we should have an openjdk package set up.

# 5. Create the spring music package
Similar to the openjdk package. The output of this package should be the compiled spring-music that other jobs or packages can include.

```
bosh generate package spring-music
```

Created package skeleton
```
packages/spring-music/
├── packaging
├── pre_packaging
└── spec
```

**spec**
We're going to be building the spring-music project during the packaging script, so we need openjdk in order to build java.
Add openjdk under dependencies, and spring-music under files.
```
---
name: spring-music

dependencies:
- openjdk

files:
- spring-music/**/*
```

**packaging**
We'll build the spring-music app, then copy the jar into $BOSH_INSTALL_TARGET

Since we depend on openjdk, it will be present under /var/vcap/packages/openjdk on the compilation VM.

```
# abort script on any command that exits with a non zero value
set -ex

cd spring-music
export JAVA_HOME=/var/vcap/packages/openjdk
./gradlew assemble

cp build/libs/spring-music.jar $BOSH_INSTALL_TARGET
```

**pre_packaging**
Nothing

This should be all we need for the spring-musci package.

# 6. Create the server job
A bosh job is a long running process that bosh will monitor. 
A job consists of a start and stop script, and a spec file specifying its nama, package dependencies and additional properties for that job.
BOSH will create a VM and start the job on in. It will monitor the process and restart it if it fails, or restart the VM if it crashes. 

A job can depend on packages, but not other jobs.

The server job for spring music will depend on the spring-music package, and the openjdk package.
We need the jar from spring-music, and openjdk in order to run jar files.

On the VM, the job's files will be under /var/vcap/jobs/{job name}

Generate the job skeleton
```
bosh generate job server
```
```
jobs/server/
├── monit
├── spec
└── templates
```

**spec**
```
---
name: server
templates:

packages:
```
- **packages** Similar to the package spec. List of packages the job depends on. The server will depend on the spring-music and openjdk. 

- **templates** One or more files used to start the job. Need to have a monit script bor BOSh to start/stop the job. For our server job it'll be a control script, described later.
- **properties** you can also specify a properties block for any properties you might want to pass to your job.

Update the job spec to fill in those sectons:
```
---
name: server
templates:
  ctl.erb: bin/ctl

packages:
- spring-music
- openjdk

properties:
  name:
    description: "User name"
    default: "User"
```
We set our dependencies, a property "name" with a default name.
The templates section specifies that there will be a file called ctl.erb under the `jobs/server/templates` folder, and on the VM it will be available under `/var/vcap/jobs/server/bin/ctl`

**monit** This file will contain a monit script for how to start and stop the server. 
```
check process server
  with pidfile /var/vcap/sys/run/server/pid
  start program "/var/vcap/jobs/server/bin/ctl start"
  stop program "/var/vcap/jobs/server/bin/ctl stop"
  group vcap
``` 

**template** Last thing we need to do is add a template to tell BOSH how to start/stop the job. 
Create the `jobs/server/templates/ctl.erb` file. It will be very similar to a bash script that takes in a start/stop command
```
#!/bin/bash

RUN_DIR=/var/vcap/sys/run/server
LOG_DIR=/var/vcap/sys/log/server
PIDFILE=${RUN_DIR}/pid

case $1 in

  start)
    set -ex
    mkdir -p $RUN_DIR $LOG_DIR

    chown -R vcap:vcap $RUN_DIR $LOG_DIR

    echo $$ > $PIDFILE

    cd /var/vcap/jobs/server

    exec /var/vcap/packages/openjdk/bin/java -jar ./packages/spring-music/spring-music.jar \
      --name="<%= p("name") %>" \
      >>  $LOG_DIR/server.stdout.log \
      2>> $LOG_DIR/server.stderr.log

    ;;

  stop)
    kill -9 `cat $PIDFILE`
    rm -f $PIDFILE

    ;;

  *)
    echo "Usage: ctl {start|stop}" ;;

esac
```

Script taken from the bosh docs. When called with `start` the script will execute a process and save the process id in a file. When called with `stop` the script will kill the process with that id.

The interesting part is the `exec /var/vcap/packages/openjdk/bin/java -jar  ./packages/spring-music/spring-music.jar` line, which tells the script how to start our job.

Notice that this is not quite a bash script, but is a template. BOSH will do a pass on this template, and replace any placeholders with actual values. 
In the line `--name="<%= p("name") %>"` BOSH will look for a property called "name", and replace it with its actual value.

After adding the ctl script we should be done with setting up the job.

BREAK POINT

# 7. Create the release

We should have all we need to create the spring-music release. Create a dev release by running
```
bosh create release --force
```

Release name: spring-music-release-{name}

# 8. Upload the release

Connect to the bosh director set up for the workshop with the username and password
```
bosh target <ip>
```

```
bosh upload release
```

This will upload the release to the bosh director, and run the packaging scripts.

# 9. Run your spring music job

To run a bosh release you need to provide a deployment manifest. This is a yml file that tells BOSH what jobs you want to run, what kind of VM you want to have running that job, how many vms and any properties that should be passed to the job. 

A deployment can refer to one or multiple releases.

Sample manifest:
```
---
name: spring-server-{name}
director_uuid: {director uuid}

releases:
- name: spring-music-release-{name}
  version: latest

stemcells:
- alias: ubuntu
  os: ubuntu-trusty
  version: latest

update:
  canaries: 0
  max_in_flight: 1
  canary_watch_time: 1000-30000
  update_watch_time: 1000-30000

instance_groups:
- name: server
  instances: 1
  azs: [z1]
  jobs:
  - name: server
    release: spring-music-release-{name}
  vm_type: t2.small
  vm_extensions: [5GB_ephemeral_disk]
  stemcell: ubuntu
  networks:
  - name: private
  properties:
    name: ERGINNN
```
**director_uuid** uuid of a director. Needs to be specified so you don't accidentally deploy to the wrong bosh director. Get the director uuid by running `bosh status --uuid`
**name** name of this deployment. Make it unique
**releases** Releases that will be used by this deployment
**stemcell** Version of the stemcell bosh should use. Can use ` bosh stemcells` to see possible values

**update** Update behaviour for the job.
**instance groups** How the jobs should be deployed. An instance group specifies what jobs should run, the vm type, network type and additional properties for those jobs. 

You can get the vm types and network types by running `bosh cloud-config`. In this case we're using a t2.small size vm, with a 5GB disk attached to it.

Target the manifest

```
bosh deployment manifest.yml
```

Deploy
```
bosh deploy
```
