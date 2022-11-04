# 固件升级

## 如何升级

1. 通过WEB升级

2. 通过curl升级：

   > curl -k -u <username>:<password> -X POST https://<BMCIP>/redfish/v1/UpdateService/ --data-binary '@<filename>'

## 升级过程

![1667551139594](softwaremanager.assets/1667551139594.png)

## 打包格式

openbmc编译完成后，会在/openbmc/build/palos/tmp/deploy/images下产生两个可以用于升级的tar包，其中内容如下。

```shell
# tar -tf obmc-cpld.tar.gz
MANIFEST
publickey
cpld.jed
cpld.jed.sig
MANIFEST.sig
publickey.sig
# tar -xf obmc-cpld.tar.gz
# cat MANIFEST
purpose=xyz.openbmc_project.Software.Version.VersionPurpose.CPLD
version=test
MachineName=palos
KeyType=OpenBMC
HashType=RSA-SHA256
#
```

## 打包工具

```shell
$ ./gen-fw-tar -h
Generate Tarball with firmware image and MANIFEST Script

Generates a firmware image tarball from given file as input.
Creates a MANIFEST for image verification and recreation
Packages the image and MANIFEST together in a tarball

usage: gen-fw-tar [OPTION] <Firmware FILE>...

Options:
   -o, --out <file>       Specify destination file. Defaults to
                          `pwd`/obmc-$type.tar.gz if unspecified.
   -s, --sign <path>      Sign the image. The optional path argument specifies
                          the private key file. Defaults to the bash variable
                          PRIVATE_KEY_PATH if available, or else uses the
                          open-source private key in this script.
   -m, --machine <name>   Optionally specify the target machine name of this
                          image.
   -v, --version <name>   Specify the version of bios image file
   -t, --type <name>      Specify the type of fw image file
   -h, --help             Display this help text and exit.

$ ./gen-fw-tar -s -m palos -v test -t cpld cpld.jed
Image is NOT secure!! Signing with the open private key!
Creating MANIFEST for the image
MANIFEST
publickey
cpld.jed
cpld.jed.sig
MANIFEST.sig
publickey.sig
Firmware image tarball is at /home/workspace/openbmc/build/palos/workspace/sources/phosphor-software-manager/obmc-cpld.tar.gz
$
```

