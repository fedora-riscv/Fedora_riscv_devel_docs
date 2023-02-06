## Fedora 模块性（Modularity）

简而言之，Modularity 可以扩展具有以下功能的 RPM 存储库： 
* 模块流（Module Stream）添加具有相同名称但从不同来源和不同构建配置构建的其他包。 
* 模块流可以有不同的生命周期。  例如，分发包中的包可以具有与分发包的生命周期不同的生命周期。 
* 模块流可以具有其组件的不同安装配置文件，即您可以定义一个或多个模块流配置文件，其中将包含应一起安装的 RPM 组。 

## Fedora 模块构建服务

Fedora 模块构建服务（Module Build Service, MBS）协调模块的构建过程，并负责以下任务：
• 为客户端工具提供交互接口，以便提交模块和查询构建状态
• 验证输入的数据（比如modulemd、RPM SPEC等）是可用且正确的
• 在支持的构建系统（比如Koji）内准备构建环境
• 安排并构建模块组件，跟踪构建状态
• 在状态改变时发送总线（比如fedmsg、Apache MQ）信息，以便其他基础设施服务能够获取任务

Rocky Linux提供了一个不太完全的mbs安装指南：https://wiki.rockylinux.org/archive/legacy/mbs_installation/

## 本地构建模块
有关在线（貌似需要Fedora的有权限的ID）构建的教程，请参见：https://docs.fedoraproject.org/en-US/modularity/building-modules/fedora

## 先决条件 

在开始本地构建模块之前，必须安装以下依赖项： 
$ sudo dnf install module-build-service fedpkg rpkg如果您没有打算以 root 身份运行构建命令，则必须将您的用户添加到 mock用户组： 
$ sudo usermod -a -G mock USERNAME
$ newgrp从 distgit 构建你的模块 
本地构建开始使用 fedpkg从你的 dist-git 仓库中。 
例如，提交一个构建 testmodule:master模块，运行： 

$ fedpkg clone modules/testmodule
$ cd testmodule
$ git checkout master
$ fedpkg module-build-local

构建完成后，您将能够在~/modulebuild/builds/MODULE/results/ 找到构建结果。

另外，本地构建不支持流扩展。  如果您的模块依赖于另一个模块的多个流，例如 platform，您需要指定要构建的流。  例如： fedpkg module-build-local -s platform:f28.

##  部署 MBS 

### 安装软件包

主要是安装 Apache 以及 MBS 的依赖。

```
dnf install fedmsg python3-gssapi git httpd mod_ssl python3-mod_wsgi python3-solv python3-pungi python3-psycopg2 mod_auth_gssapi module-build-service -y
```

### 配置并启动 Fedmsg

Fedmsg 是 Fedora 自己的一个消息服务，MBS 会使用 Fedmsg 在 Fedora 的各个基础设施之间相互协同。但我们不需要 Fedmsg 来协同，所以目前的配置只是运行 Fedmsg，但是并不向 Fedora 发送数据。因此需要改动一下默认配置，主要是注释掉 Fedora 的服务器与签名验证：

```python
/etc/fedmsg.d/endpoints.py
------
    endpoints={
        # These are here so your local box can listen to the upstream
        # infrastructure's bus.  Cool, right?  :)
        "fedora-infrastructure": [
            # "tcp://hub.fedoraproject.org:9940", <- 注释此行
            # "tcp://stg.fedoraproject.org:9940",
        ],
        # "debian-infrastructure": [
        #    "tcp://fedmsg.olasd.eu:9940",
        # ],

```

```python
/etc/fedmsg.d/module_build_service.py
------
config = {
    # Just for dev.
    "validate_signatures": False,  # <- 设置为 False
	  'sign_messages': False,   # <- 设置为 False
    # Talk to the relay, so things also make it to composer.stg in our dev env
    "active": True,
    # Since we're in active mode, we don't need to declare any of our own
    # passive endpoints.  This placeholder value needs to be here for the tests
    # to pass in Jenkins, though.  \o/
    "endpoints": {
        "fedora-infrastructure": [
            # Just listen to staging for now, not to production (spam!)
            # "tcp://hub.fedoraproject.org:9940", # <- 注释掉
            # "tcp://stg.fedoraproject.org:9940"
        ]
    },
    # Start of code signing configuration
    # 'sign_messages': True,  # <- 注释掉
    # 'validate_signatures': True,  # <- 注释掉
    # 'crypto_backend': 'x509',
    .....
}

```

```python
/etc/fedmsg.d/ssl.py
------
config = dict(
    sign_messages=False,   # <- 设置为 False
    validate_signatures=False,   # <- 设置为 False
    ...)
```

这里的 Topic 也要改为自己的：

```python
/etc/fedmsg.d/base.py
------
config = dict(
    # Prefix for the topic of each message sent.
    topic_prefix="org.fedora-plct", # <- 改为自己的名称

    # Set this to dev if you're hacking on fedmsg or an app.
    # Set to stg or prod if running in the Fedora Infrastructure
    environment="dev",
    ......)
```

用 Systemd 启动这两个服务：

```bash
systemctl enable fedmsg-hub --now
systemctl enable fedmsg-relay --now
```

### mbs-frontend 的 Apache 配置

MBS 的前端是一个 wsgi 应用，不能直接运行，而应该配合 Apache Httpd 这样的服务器。新建以下配置文件：

```txt
/etc/httpd/conf.d/mbs.conf
------
<IfModule mod_ssl.c>
<VirtualHost *:443>
  ServerName mbs.gnulab.org
  WSGIDaemonProcess mbs user=mbs group=mbs threads=5
    WSGIScriptAlias / /etc/module-build-service/mbs.wsgi
  WSGIPassAuthorization on
    <Directory /etc/module-build-service>
        WSGIProcessGroup mbs
        WSGIApplicationGroup %{GLOBAL}
        Require all granted
    </Directory>
#    <Location />  # <- 测试时不使用 Krb 注释掉
#        AuthType GSSAPI
#        AuthName "GSSAPI Single Sign On Login"
#        GssapiCredStore keytab:/etc/koji.keytab
#        Require valid-user
#    </Location>

# 测试时不使用 SSL 注释掉
# SSLCertificateFile /etc/letsencrypt/live/mbs.gnulab.org/fullchain.pem 
# SSLCertificateKeyFile /etc/letsencrypt/live/mbs.gnulab.org/privkey.pem
# Include /etc/letsencrypt/options-ssl-apache.conf
</VirtualHost>
</IfModule>
```

也要新建 WGSI 文件：

```python
/etc/module-build-service/mbs.wsgi
------
import logging
logging.basicConfig(level=logging.DEBUG)
from module_build_service import app as application
```

### MBS 配置

接下来，我们为 MBS 新建用户并配置文件权限：

```
useradd mbs
passwd mbs
chown -R mbs:mbs /etc/module-build-service/
touch /tmp/module_build_service.log
chown mbs:fedmsg /tmp/module_build_service.log
chmod 664 /tmp/module_build_service.log
```

然后在 Koji 系统内添加 MBS 用户：

```
koji add-user mbs
koji call addBType module
koji grant-cg-access mbs module-build-service --new
```

MBS 有自己的 Koji 配置，按照实际情况来写：

```ini
/etc/module-build-service/koji.conf
------
[koji]

;url of XMLRPC server
server = https://openkoji.iscas.ac.cn/kojihub
;url of web interface
weburl = https://openkoji.iscas.ac.cn/koji
;url of package download site
topurl = https://openkoji.iscas.ac.cn/kojifiles
;path to the koji top directory
topdir = /mnt/koji

; configuration for SSL athentication
authtype = ssl

;client certificate
cert = /home/mbs/.koji/mbs.pem

;certificate of the CA that issued the HTTP server certificate
serverca = /home/mbs/.koji/koji_ca_cert.crt
```

最后是 MBS 本体的配置，按需修改：

```python
/etc/module-build-service/config.py
------
# -*- coding: utf-8 -*-
# SPDX-License-Identifier: MIT
from os import environ, path

# FIXME: workaround for this moment till confdir, dbdir (installdir etc.) are
# declared properly somewhere/somehow
confdir = path.abspath(path.dirname(__file__))
# use parent dir as dbdir else fallback to current dir
dbdir = path.abspath(path.join(confdir, "..")) if confdir.endswith("conf") else confdir


class ProdConfiguration(object):
    DEBUG = True
    # Make this random (used to generate session keys)
    SECRET_KEY = "74d9e9f9cd40e66fc6c4c2e9987dce48df3ce98542529126"
    SQLALCHEMY_DATABASE_URI = "sqlite:///{0}".format(path.join(dbdir, "module_build_service.db")) # 测试期间就用 SQLite 了
    #SQLALCHEMY_DATABASE_URI = 'postgresql://mbs:mysupersecretepasswordmbs@koji.gnulab.org/mbs'
    SQLALCHEMY_TRACK_MODIFICATIONS = True
    # Where we should run when running "manage.py run" directly.
    HOST = "0.0.0.0"
    PORT = 5000

    # Global network-related values, in seconds
    NET_TIMEOUT = 120
    NET_RETRY_INTERVAL = 30

    #DISTGITS = {"git+https://git.centos.org": ("git clone {repo_path}", "get_sources.sh")}
    SYSTEM = "koji"
    MESSAGING = "fedmsg"  # or amq
    MESSAGING_TOPIC_PREFIX = ["org.fedora-plct"]  # 修改为与 Fedmsg 配置一致
    KOJI_CONFIG = "/etc/module-build-service/koji.conf"
    KOJI_PROFILE = "koji"
    ARCHES = ["riscv64"]
    ALLOW_ARCH_OVERRIDE = False
    KOJI_REPOSITORY_URL = "https://openkoji.iscas.ac.cn/kojifiles"
    KOJI_TAG_PREFIXES = ["module", "scrmod"]
    KOJI_ENABLE_CONTENT_GENERATOR = True
    CHECK_FOR_EOL = False
    PDC_URL = "https://pdc.fedoraproject.org/rest_api/v1"
    PDC_INSECURE = False
    PDC_DEVELOP = True
    SCMURLS = ["https://src.fedoraproject.org"]
    YAML_SUBMIT_ALLOWED = True

    # How often should we resort to polling, in seconds
    # Set to zero to disable polling
    POLLING_INTERVAL = 600

    # Determines how many builds that can be submitted to the builder
    # and be in the build state at a time. Set this to 0 for no restrictions
    NUM_CONCURRENT_BUILDS = 5

    ALLOW_CUSTOM_SCMURLS = False

    RPMS_DEFAULT_REPOSITORY = " git+https://src.fedoraproject.org/rpms/"
    RPMS_ALLOW_REPOSITORY = True
    #RPMS_DEFAULT_CACHE = "http://pkgs.fedoraproject.org/repo/pkgs/"
    RPMS_ALLOW_CACHE = False

    MODULES_DEFAULT_REPOSITORY = "git+https://src.fedoraproject.org/modules/"
    MODULES_ALLOW_REPOSITORY = False
    MODULES_ALLOW_SCRATCH = True
    ALLOW_ONLY_COMPATIBLE_BASE_MODULES = True

    ALLOWED_GROUPS_TO_IMPORT_MODULE = set()

    # Available backends are: console and file
    LOG_BACKEND = "file"

    # Path to log file when LOG_BACKEND is set to "file".
    LOG_FILE = "/tmp/module_build_service.log"

    # Available log levels are: debug, info, warn, error.
    LOG_LEVEL = "debug"

    # Allow stream override
    ALLOW_STREAM_OVERRIDE_FROM_SCM = True

    # Settings for Kerberos
    KRB_KEYTAB = "/etc/mbs.keytab"
    KRB_PRINCIPAL = "mbs@GNULAB.ORG"

    # AMQ prefixed variables are required only while using 'amq' as messaging backend
    # Addresses to listen to
    AMQ_RECV_ADDRESSES = [
        "amqps://messaging.mydomain.com/Consumer.m8y.VirtualTopic.eng.koji",
        "amqps://messaging.mydomain.com/Consumer.m8y.VirtualTopic.eng.module_build_service",
    ]
    # Address for sending messages
    AMQ_DEST_ADDRESS = \
        "amqps://messaging.mydomain.com/Consumer.m8y.VirtualTopic.eng.module_build_service"
    AMQ_CERT_FILE = "/etc/module_build_service/msg-m8y-client.crt"
    AMQ_PRIVATE_KEY_FILE = "/etc/module_build_service/msg-m8y-client.key"
    AMQ_TRUSTED_CERT_FILE = "/etc/module_build_service/Root-CA.crt"

    # Disable Client Authorization
    NO_AUTH = True  # 测试或者内部使用可以关闭认证
    AUTH_METHOD = "kerberos"
    LDAP_URI = "ldap://koji.gnulab.org"
    LDAP_GROUPS_DN = "ou=group,dc=gnulab,dc=org"
    ADMIN_GROUPS = {"packageradmin"}
    ALLOWED_GROUPS = {"packager"}
    KOJI_CG_DEVEL_MODULE = True
    KOJI_PROXYUSER = True
    REBUILD_STRATEGY = 'only-changed'
    REBUILD_STRATEGY_ALLOW_OVERRIDE = True
    KOJI_CG_BUILD_TAG_TEMPLATE = "{}-modular-updates-candidate"
    KOJI_CG_DEFAULT_BUILD_TAG = "modular-updates-candidate"
    # Extra options set for newly created Koji tags
    KOJI_TAG_EXTRA_OPTS = {
        "mock.package_manager": "dnf",
        # This is needed to include all the Koji builds (and therefore
        # all the packages) from all inherited tags into this tag.
        # See https://pagure.io/koji/issue/588 and
        # https://pagure.io/fm-orchestrator/issue/660 for background.
        "repo_include_all": True,
        # Has been requested by Fedora infra in
        # https://pagure.io/fedora-infrastructure/issue/7620.
        # Disables systemd-nspawn for chroot.
        "mock.new_chroot": 0,
        # Works around fail-safe mechanism added in DNF 4.2.7
        # https://pagure.io/fedora-infrastructure/issue/8410
        "mock.yum.module_hotfixes": 1,
    }
    # DEFAULT_DIST_TAG_PREFIX = 'module_'

```

```
systemctl restart httpd
su mbs # 切换用户，防止文件权限问题
mbs-manager db upgrade
```

### 测试和使用 MBS

在使用 MBS 之前，需要导入平台模块。

```
su mbs
```

这是一个平台模块的示例

```
/home/mbs/platform-riscv-test.yaml 
------
document: modulemd
version: 1
data:
    name: platform
    stream: f37.0.0
    version: 3
    context: 00000000
    summary: Testing Rust dev base
    description: Testing Rust dev base
    license:
        module: [MIT]
    profiles:
        buildroot:
            rpms: [bash, bzip2, coreutils, cpio, diffutils, findutils, gawk, gcc, gcc-c++, grep, gzip, info, make, module-build-macros, patch,
redhat-rpm-config, rpm-build, sed, shadow-utils, tar, unzip, util-linux, which, xz]
        srpm-buildroot:
            rpms: [bash, gnupg2, module-build-macros, redhat-rpm-config, rpm-build, shadow-utils]
    xmd:
        mbs:
            buildrequires: {}
            commit: virtual
            requires: {}
            koji_tag: f37-build-side-32-misc-devel
            mse: TRUE
            virtual_streams: [f37]
```

然后我们用命令行管理工具导入：

```
mbs-manager import_module /home/mbs/platform-riscv-test.yaml 
```

模块的定义是一个 YAML 文件，下面就是一个示例：

```yaml
---
document: modulemd
version: 2
data:
  name: zlib
  stream: '1.2.12'
  summary: zlib
  description: >
      zlib.
  license:
    module:
      - MIT
  dependencies:
  - buildrequires:
      platform: []
    requires:
      platform: []
  components:
    rpms:
      zlib:
        rationale: zlib
        ref: f37
        buildorder: 0
...
```

使用 Wegt 发送该 YAML 到 MBS 服务器上面。

```
wget --quiet \
  --method POST \
  --header 'Content-Type: multipart/form-data; boundary=---011000010111000001101001' \
  --body-data '-----011000010111000001101001\r\nContent-Disposition: form-data; name="modulemd"\r\n\r\n---\ndocument: modulemd\nversion: 2\ndata:\n  name: zlib\n  stream: '\''1.2.12'\''\n  summary: zlib\n  description: >\n      zlib.\n  license:\n    module:\n      - MIT\n  dependencies:\n  - buildrequires:\n      platform: []\n    requires:\n      platform: []\n  components:\n    rpms:\n      zlib:\n        rationale: zlib\n        ref: f37\n        buildorder: 0\n...\r\n-----011000010111000001101001--\r\n' \
  --output-document \
  - 'http://127.0.0.1:18080/module-build-service/1/module-builds/?='
```
