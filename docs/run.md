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
cd ebpf/libbpf/src
sudo make install
cd -
sudo ./loxilb 
```
* To run with integrated api-server, we can use the following :

```
./loxilb --tls-key=api/certification/server.key --tls-certificate=api/certification/server.crt --host=0.0.0.0 --port=11111 --tls-port=8091 -a
```

## From docker (easy)

* Get the loxilb official docker image 

```
docker pull loxilbio/loxilb:beta
```

* To run loxilb docker, we can use the following commands :

```
docker run -u root --cap-add SYS_ADMIN   --restart unless-stopped --privileged -dit -v /dev/log:/dev/log --entrypoint /root/loxilb-io/loxilb/loxilb --name loxilb loxilbio/loxilb:beta
```
OR

```
docker run -u root --cap-add SYS_ADMIN   --restart unless-stopped --privileged -dit -v /dev/log:/dev/log --name loxilb loxilbio/loxilb:beta
docker exec -it loxilb bash
cd /root/loxilb-io/loxilb
./loxilb 
```
The latter option is useful for those who are open to experiment/explore

