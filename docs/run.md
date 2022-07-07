# loxilb - How to build/run

## Right from code (difficult)

* Build custom iproute2 package 

```
git clone https://github.com/loxilb-io/iproute2.git
cd iproute2
cd libbpf/src/
mkdir build
DESTDIR=build make install
cd ../../
export PKG_CONFIG_PATH=$PKG_CONFIG_PATH:`pwd`/libbpf/src/
LIBBPF_FORCE=on LIBBPF_DIR=`pwd`/libbpf/src/build ./configure
make
sudo cp -f tc/tc /usr/local/sbin/ntc
```

* Build and run loxilb 

```
git clone https://github.com/loxilb-io/loxilb.git
cd loxilb
./ebpf/utils/mkllb_bpffs.sh
make
sudo ./loxilb 

```

## From docker (easy)

* Get the loxilb official docker image 

```
docker pull loxilbio/loxilb:beta
docker run -u root --cap-add SYS_ADMIN   --restart unless-stopped --privileged -dit -v /dev/log:/dev/log --name loxilb loxilbio/loxilb:beta
```

* Run loxilb 
```
docker exec -it loxilb bash
cd /root/loxilb-io/loxilb
./loxilb 
```

