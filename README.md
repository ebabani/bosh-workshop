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

