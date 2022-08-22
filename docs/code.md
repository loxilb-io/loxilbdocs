# loxilb is organized as below:
```
├── api
│  ├── certification
│  ├── cmd
│  │  ├── loxilb-rest-api-server
│  ├── models
│  ├── restapi
│      ├── handler
│      ├── operations
├── common
├── ebpf
│  ├── common
│  ├── headers
│  │  ├── linux
│  ├── kernel
│  ├── libbpf
│  │  ├── include
│  │  │  ├── asm
│  │  │  ├── linux
│  │  │  ├── uapi
│  │  │      ├── linux
│  │  ├── scripts
│  │  ├── src
│  │  │  ├── build
│  │  │  │  ├── usr
│  │  │  │      ├── include
│  │  │  │      │  ├── bpf
│  │  │  │      ├── lib64
│  │  │  │          ├── pkgconfig
│  │  │  ├── sharedobjs
│  │  │  ├── staticobjs
│  │  ├── travis-ci
│  │      ├── managers
│  │      ├── vmtest
│  │          ├── configs
│  │              ├── blacklist
│  │              ├── whitelist
│  ├── utils
├── loxinet
├── options
├── loxilib
```

## api
This directory contains source code to host api server to handle CCM configuration requests.

## common
Common api to configure which are exposed by loxinet are defined in this directory.

## loxinet
This module implements the glue layer or the middle layer between eBPF datapath module and api modules. 
It defines functions for configuring networking and load balancing rules in the eBPF datapath.

## ebpf
This directory contains source code for loxilb eBPF datapath.

## options
This directory contains files for managing the command line options.

## loxilib
[This package contains common routines for logging, statistics and other utilities.
](https://github.com/loxilb-io/loxilib)

