---
title: Security Boot开启下CentOS8内核模块签名及加载
date: 2021-10-17 23:59:32
tags:
- Kernel
categories: Kernel
---
`Security Boot`机制是`UEFI`的一个特性，用于确保固件所加载的代码是可信的。它通过将`可信根`存放在固件中，在加载阶段校验所加载的二进制可信。当前实现主要是基于`X.509`证书的公私钥体系。固件使用可信的公钥来校验所加载的`bootloader`, `bootloader`再来校验内核或者第二阶段的`bootloader`, 内核校验所加载的内核模块，这样一级级完成校验。

在开启了`Security Boot`机制的CentOS8上，加载我们未签名的内核模块会失败:
```
[root@localhost virtualdev]# insmod ./virtualdev.ko
insmod: ERROR: could not insert module ./virtualdev.ko: Required key not available
```

接下来我们来通过签名我们的内核模块完成加载。要签名和校验我们的内核模块，我们需要一对公钥私钥。私钥用于签名，公钥需要加载到系统固件的`MOK: Machine Owner Key`列表中用于校验被签名的模块。

<!--more-->

如果本身已经具备了公私密钥对，以下生成密钥对的步骤可以跳过。

首先，创建密钥对配置文件`keypair.conf`，按需要填充相应信息:
```ini
[ req ]
default_bits = 4096
distinguished_name = req_distinguished_name
prompt = no
string_mask = utf8only
x509_extensions = fgexts

[ req_distinguished_name ]
countryName = CN
stateOrProvinceName = Beijing
localityName = Beijing
0.organizationName = just4coding
commonName = Flygoast Signing Key
emailAddress = "flygoast@126.com"

[ fgexts ]
basicConstraints=critical,CA:FALSE
keyUsage=digitalSignature
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid
```

调用`openssl`生成密钥对：
```bash
openssl req -x509 -new -nodes -utf8 -sha256 -days 36500 \
    -batch -config flygoast.config -outform DER \
    -out flygoast.der \
    -keyout flygoast.priv
```

公钥为`flygoast.der`文件, 私钥为`flygoast.priv`文件。注意:私钥文件一定要注意安全保存。

接下来需要将公钥注册到目标系统的`MOK`中。

首先，使用`mokutil`创建一个`MOK`导入请求, 这会要求输入和确认一个密码，后续步骤会需要这个密码:
```bash
mokutil --import flygoast.der
```

这时可以查看将要注册的密钥:
```plain
[root@localhost sign]# mokutil --list-new
[key 1]
SHA1 Fingerprint: 12:b6:98:48:1c:c0:ab:22:f2:45:93:19:0d:33:4c:8e:7c:7d:09:c5
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            2d:72:32:36:f7:8b:6c:32:83:1e:b8:1b:aa:1c:fc:6a:57:3b:23:34
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=CN, ST=Beijing, L=Beijing, O=just4coding, CN=Flygoast Signing Key/emailAddress=flygoast@126.com
        Validity
            Not Before: Oct 15 07:39:58 2021 GMT
            Not After : Sep 21 07:39:58 2121 GMT
        Subject: C=CN, ST=Beijing, L=Beijing, O=just4coding, CN=Flygoast Signing Key/emailAddress=flygoast@126.com
...
```

注册请求创建完成需要重启目标系统。在重启时`shim.efi`发现有注册请求，加载`MokManager.efi`去通过`UEFI` `console`界面完成注册操作。

重启后系统会进入`MokManager`的界面, 如图:
{% img /images/2021-10-17/1.png %}

选择`Enroll MOK`, 进入`Enroll MOK`界面:
{% img /images/2021-10-17/2.png %}

然后选择`View Key 0`，确认该Key是你所指定的Key, 然后回到上一界面选择`continue`:
{% img /images/2021-10-17/3.png %}

接着选择`Yes`确认注册该Key, 此时会要求输入之前创建注册请求时所登记的密码:
{% img /images/2021-10-17/4.png %}

接着选择`Reboot`:
{% img /images/2021-10-17/5.png %}


重启后，我们登录目标系统。接下来完成签名。

内核自带了工具`sign-file`用于签名内核模块:
```bash
/usr/src/kernels/$(uname -r)/scripts/sign-file sha256 flygoast.priv flygoast.der virtualdev.ko
```

签名完成后查看模块信息, 可以看到签名相关信息:
```plain
[root@localhost sign]# modinfo ./virtualdev.ko
filename:       /root/sign/./virtualdev.ko
license:        GPL
rhelversion:    8.4
srcversion:     25335292C77814400BF5B4C
depends:
name:           virtualdev
vermagic:       4.18.0-305.3.1.el8.x86_64 SMP mod_unload modversions
sig_id:         PKCS#7
signer:         Flygoast Signing Key
sig_key:        2D:72:32:36:F7:8B:6C:32:83:1E:B8:1B:AA:1C:FC:6A:57:3B:23:34
sig_hashalgo:   sha256
signature:
...
```

这时我们再次加载内核模块, 可以加载成功:
```plain
[root@localhost sign]# insmod ./virtualdev.ko
[root@localhost sign]# lsmod |grep virtualdev
virtualdev             16384  0
```

参考链接:

* https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/managing_monitoring_and_updating_the_kernel/signing-kernel-modules-for-secure-boot_managing-monitoring-and-updating-the-kernel
* https://gloveboxes.github.io/Ubuntu-for-Azure-Developers/docs/signing-kernel-for-secure-boot.html
