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
