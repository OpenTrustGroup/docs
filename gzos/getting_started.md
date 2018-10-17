# Quick Setup Guide

## Prerequisite

The development environment only support Linux for now. If your host OS is Windows or macOS, please use [GzOS development docker](https://github.com/OpenTrustGroup/gzos_dev_docker).

The development environment not only support GzOS, but also support [Google Trusty TEE](https://source.android.com/security/trusty).

## Dev Environment Setup

### Download Source Code

```shell
curl -s https://raw.githubusercontent.com/OpenTrustGroup/scripts/gzos/bootstrap | bash -s gzos [stable|dev]

alias otg='fx vendor otg'
cd gzos_stable # (or cd gzos_dev)
```
**Notes:** gzos_dev is ONLY used for code base upgrading, please do not do other works base on this environment.

### Build All

#### GzOS

```shell
otg set gztee --ree linux # select GzOS as TEE OS and Linux as REE OS (default REE OS is Fuchsia)
otg full-build [-c] # '-c': clean and build
```
#### Trusty TEE

```shell
otg set trusty # select Trusty as TEE OS (default REE OS is Linux)
otg full-build [-c] # '-c': clean and build
```


### Run On QEMU

```shell
otg run-qemu
```

### Run CI Test

```shell
 # run all test programs
otg run-qemu -c all

# run test programs matching the filter pattern
otg run-qemu -c <filter pattern>

# dump test logs after test finished
cat <expect.log | serial0.log | serial1.log>
```

### Show Backtrace (Only For GzOS)

If kernel or application crashed, the following command can translate the backtrace raw info
to show the corresponding source file and line number

```shell
otg backtrace [log_file]
```
