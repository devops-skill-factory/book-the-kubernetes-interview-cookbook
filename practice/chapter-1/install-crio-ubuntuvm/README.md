# Install CRI-O Container Runtime on Ubuntu VM

## Prepare VM

1. Install [VirtualBox](https://www.virtualbox.org/wiki/Downloads)
2. Install [Vagrant](https://developer.hashicorp.com/vagrant/install)
3. Spin up your VirtualBox VM:
```bash
vagrant up
```
VirtualBox config is defined via `Vagrantfile`. To find more available Vagrant Box Images, see [portal.cloud.hashicorp.com/vagrant/discover](https://portal.cloud.hashicorp.com/vagrant/discover) 

4. SSH into VM:
```bash
vagrant ssh
```
5. Get root privileges:
```bash
sudo -i
```

## Install CRI-O

⚠️ **NOTE**  
> We have provisioned Ubuntu Focal Fossa 20.04.  
> Keep in mind that CRI-O installation varies between different Ubuntu versions.

[Install CRI-O](https://github.com/cri-o/cri-o/blob/main/install.md#debian---raspbian---ubuntu):
1. Add third-party Kubic repository dependencies:
```bash
# Add the Kubic repository specifically built for Focal (xUbuntu_20.04)
echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_20.04/ /" | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list

# Import the proper repository release key
curl -L "https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_20.04/Release.key" | sudo apt-key add -

# Refresh your package index so apt catches the new source
sudo apt-get update

# Run the installation 
sudo apt-get install -y containers-common cri-o-runc
```
2. Install required dependencies for `cri-o`:
```bash
apt-get update -qq && apt-get install -y libbtrfs-dev containers-common git libassuan-dev libglib2.0-dev libc6-dev libgpgme-dev libgpg-error-dev libseccomp-dev libsystemd-dev libselinux1-dev pkg-config go-md2man cri-o-runc libudev-dev software-properties-common gcc make
```

Install `crun` dependency:
```bash
apt-get install -y crun
```

Install `conmon` dependency:
```bash
apt-get install -y conmon
```

3. Install [Go](https://go.dev/doc/install):
```bash
# Download Go 1.26.3
curl -LO https://go.dev/dl/go1.26.3.linux-amd64.tar.gz

# Remove any previous Go installation 
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.26.3.linux-amd64.tar.gz

# Add /usr/local/go/bin to the PATH environment variable
export PATH=$PATH:/usr/local/go/bin

# Check go installation
go version
```
3. Clone the `cri-o` source code:
```bash
git clone https://github.com/cri-o/cri-o # or your fork
cd cri-o
```
4. To install with default buildtags using seccomp, use:
```bash
make
make install
```
Otherwise, if you do not want to build CRI-O with seccomp support you can add BUILDTAGS="" when running make.
```bash
make BUILDTAGS=""
make install
```

For available tags, see [build tags](https://github.com/cri-o/cri-o/blob/main/install.md#build-tags).

5. When you install `cri-o` for the first time, you can generate and install configuration files with:
```bash
make install.config
```
6. Edit `vim /etc/containers/registries.conf` and verify that the registries option has valid values in it. For example:
```bash
[registries.search]
registries = ['registry.access.redhat.com', 'registry.fedoraproject.org', 'quay.io', 'docker.io']

[registries.insecure]
registries = []

[registries.block]
registries = []
```
7. Start CRI-O:
```bash
make install.systemd
```
8. Let `systemd` take care of running CRI-O:
```bash
systemctl daemon-reload
systemctl enable crio
systemctl start crio
```
9. Check `cri-o` installation:
```bash
crio --version
```

## Check CRI-O with crictl

To work with `cri-o` runtime install `crictl` CLI. For more information, see [Install crictl](https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md#install-crictl).
```bash
# Define crictl version
VERSION="v1.32.0"

# Download crictl
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz

# Untar crictl 
tar -C /usr/local/bin -xzf crictl-$VERSION-linux-amd64.tar.gz
```

Verify `crictl` installation:
```bash
crictl info
```

In the output you should see that `crictl` configured to use `cri-o`:
```bash
...
"config": {
    "crio": {
      "AbsentMountSourcesToReject": null,
      "AddInheritableCapabilities": false,
      "AdditionalArtifactStores": null,
      "AdditionalDevices": null,
      "AllowedDevices": [
        "/dev/fuse",
        "/dev/net/tun"
      ],
...
```

Pull test Nginx Docker image:
```bash
crictl pull docker.io/library/nginx:latest
```

Verify that image was downloaded:
```bash
crictl images
```

## Vagrant Commands

- To destroy VM:
```bash
vagrant destroy
```

- To pause VM:
```bash
vagrant halt
```