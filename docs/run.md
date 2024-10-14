# loxilb - How to build/run

## 1. Build from code and run (difficult)

* Install GoLang > v1.17

```
wget https://go.dev/dl/go1.22.0.linux-amd64.tar.gz && sudo tar -xzf go1.22.0.linux-amd64.tar.gz --directory /usr/local/
export PATH="${PATH}:/usr/local/go/bin"
```
arm64 users, use the command below.
```
wget https://go.dev/dl/go1.22.0.linux-arm64.tar.gz && sudo tar -xzf go1.22.0.linux-arm64.tar.gz --directory /usr/local/
export PATH="${PATH}:/usr/local/go/bin"
```
   
* update package   
```
sudo apt update   
```
  
* Install standard packages
```
sudo apt install -y clang llvm libelf-dev gcc-multilib libpcap-dev vim net-tools linux-tools-$(uname -r) elfutils dwarves git libbsd-dev bridge-utils wget unzip build-essential bison flex iproute2 curl
```
arm64 users, use the command below.
```
 apt-get install -y clang llvm libelf-dev libpcap-dev vim net-tools ca-certificates elfutils dwarves git libbsd-dev bridge-utils wget unzip build-essential bison flex sudo iproute2 pkg-config tcpdump iputils-ping curl bash-completion
```

* Install loxilb eBPF loader tools
```
curl -sfL https://github.com/loxilb-io/tools/raw/main/loader/install.sh | sh -
```
* Install Openssl 3
```
wget https://github.com/openssl/openssl/releases/download/openssl-3.3.1/openssl-3.3.1.tar.gz   
tar -xvzf openssl-3.3.1.tar.gz   
cd openssl-3.3.1
./Configure enable-ktls '-Wl,-rpath,$(LIBRPATH)' --prefix=/usr/local/build   
make -j$(nproc)
sudo make install_dev install_modules
cd -   
sudo cp -a /usr/local/build/include/openssl /usr/include/   
sudo bash -c "if [ -d /usr/local/build/lib64  ] ; then mv /usr/local/build/lib64  /usr/local/build/lib; fi"   
sudo cp -fr /usr/local/build/lib/* /usr/lib/   
sudo ldconfig   
rm -fr openssl-3.3.1*   
```
* Build and run loxilb 
```
git clone --recurse-submodules https://github.com/loxilb-io/loxilb.git
cd loxilb
./loxilb-ebpf/utils/mkllb_bpffs.sh
make
sudo ./loxilb 
```
* Build and use loxicmd 

```
git clone https://github.com/loxilb-io/loxicmd.git
cd loxicmd
go get .
make
sudo cp -f loxicmd /usr/local/sbin/
```
loxicmd usage guide can be found [here](https://loxilb-io.github.io/loxilbdocs/cmd/)

## 2. Build and run using docker (easy)

Build the docker image    
```
git clone --recurse-submodules https://github.com/loxilb-io/loxilb.git
cd loxilb
make docker
```

This would create the docker image ```ghcr.io/loxilb-io/loxilb:latest``` locally. One can then run loxilb in standalone mode by following guide [here](https://github.com/loxilb-io/loxilbdocs/blob/main/docs/standalone.md)


## 3. Running in Kubernetes   
* For running in K8s environment, kindly follow [kube-loxilb](https://loxilb-io.github.io/loxilbdocs/kube-loxilb/) guide     

