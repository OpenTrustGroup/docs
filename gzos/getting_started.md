# Quick Setup Guide

The development environment is not only for GzOS, but also for Trusty TEE.
Future plan is to support OP-TEE. 

* [Go To Trusty Setup](#trusty)

## GzOS

### Check out source code

```shell
curl -s https://raw.githubusercontent.com/OpenTrustGroup/scripts/gzos/bootstrap | \
    bash -s gzos [stable|dev]
```
**Notes:** gzos_dev is ONLY used for code base upgrading, please do not do other works base on this environment.


### Build GzOS

```shell
cd gzos_stable # (or cd gzos_dev)
fx set gztee
fx full-build
```
**Notes:** For now GzOS only can be used as an TEE OS and running in ARM TrustZone.
In future, GzOS will support other execution environment, e.g. as a 
lightweight passive hypervisor in ARMv8 EL2.

### Clean Build

```shell
fx clean-build gztee
```
**Notes:** "fx clean build <project_name>" is equal to "fx set <project_name> && fx full-build"


### Start QEMU

```shell
fx run-qemu
```

### Run CI test

```shell
 # run all test programs
fx run-qemu -c all

# run test programs matching the filter pattern
fx run-qemu -c <filter pattern>

# dump test logs after test finished
cat <expect.log | serial0.log | serial1.log>
```

### Show Backtrace

If kernel or application crashed, the following command can translate the backtrace raw info
to show the corresponding source file and line number

```shell
fx backtrace <log_file>
```

## Trusty

```shell
# Check out source code
curl -s https://raw.githubusercontent.com/OpenTrustGroup/scripts/gzos/bootstrap | bash -s trusty
    
# Build Trusty
cd trusty
fx set arm64
fx full-build

# Start QEMU
fx run-qemu

# Run test
fx run-qemu -c all
```