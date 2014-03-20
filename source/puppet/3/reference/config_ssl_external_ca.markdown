---
layout: default
title: "SSL Configuration: External CA Support"
canonical: "/puppet/latest/reference/config_ssl_external_ca.html"
---

[conf]: /guides/configuring.html#puppetconf
[verify_header]: /references/latest/configuration.html#sslclientverifyheader
[client_header]: /references/latest/configuration.html#sslclientheader
[ca_auth]: /references/latest/configuration.html#sslclientcaauth
[puppetdb]: /puppetdb/latest

Starting with **Puppet 3.2.0 and later,** Puppet can use an existing external CA for all of its SSL communications. This page describes the supported configurations for external CAs.

自**Puppet 3.2.0开始**，Puppet可以使用已有的外部证书颁发机构(CA)进行SSL通信。本页描述了用于支持外部证书颁发机构(CA)的配置。

> **Note:** This page uses RFC 2119 style semantics for MUST, SHOULD, MAY.

> **注意：** 对于必须、应当、可能，本页面使用RFC2119的语义格式。

> **Historical note:** Using an external CA with Puppet has sometimes worked (or appeared to), but it frequently broke, and there was never a concerted effort to keep it working. Most recently, a security fix in Puppet 2.7.18 broke nearly all external CAs, and prevented a lot of people from upgrading through the early 3.x series. (See [issue #15561](https://projects.puppetlabs.com/issues/15561).)

> **过往记录：** 在Puppet中使用外部证书颁发机构(CA)有时候可以正常使用(似乎是这样的)，但是经常出故障，并且从未有一个统一的解决方案以保证它的正常运作。最近，Puppet 2.7.18的一个安全修补程序还破坏了几乎所有的外部证书颁发机构(CA)，并且使得很多人升级至早期的3.x系统版本时失败。(参见[issue #15561](https://projects.puppetlabs.com/issues/15561)。)

> Starting with Puppet 3.2, Puppet Labs officially supports and tests external CAs in certain configurations. We're hoping that 15561 will be the last total external CA breakage. Thanks for your patience, and please get in touch if your use case isn't covered.

> 从Puppet 3.2开始，Puppet Labs官方支持并且以明确的配置测试外部证书颁发机构(CA)，我们希望issue 15561将成为最后一个外部证书颁发机构(CA)的破坏者。感谢你的耐心，如果你的使用实例并未涵盖于此，请让我们知道。


Supported External CA Configurations
-----

被支持的外部CA配置
-----

Puppet ≥ 3.2 supports _some_ external CA configurations, but not every possible arrangement. We fully support the following setups:

Puppet 3.2及以上的版本支持_一些_外部的证书颁发机构(CA)配置，但不是所有可能的情况。我们完全支持以下的设置：

1. [Single self-signed CA which directly issues SSL certificates.](#option-1-single-ca) 
2. [Single, intermediate CA issued by a root self-signed CA.](#option-2-single-intermediate-ca)  The intermediate CA directly issues SSL certificates; the root CA doesn't.
3. [Two intermediate CAs, both issued by the same root self-signed CA.](#option-3-two-intermediate-cas-issued-by-one-root-ca)
    * One intermediate CA issues SSL certificates for puppet master servers.
    * The other intermediate CA issues SSL certificates for agent nodes.
    * Agent certificates can't act as servers, and master certificates can't act as clients.

1. [直接颁发SSL证书的独立自签名证书颁发机构(CA).](#option-1-single-ca)
2. [独立的中间证书颁发机构(CA)，颁发自一个自签名的根证书颁发机构(CA)](#option-2-single-intermediate-ca) 中间证书颁发机构(CA)直接颁发SSL证书，根证书颁发机构(CA)不颁发。
3. [两个中间证书颁发机构(CA)，都由相同的自签名的根证书颁发机构(CA)所颁发](#option-3-two-intermediate-cas-issued-by-one-root-ca)
    * 一个中间机构为puppet主服务器颁发SSL证书。
    * 另一个中间机构为代理节点颁发SSL证书。
    * 代理证书不能够成为服务器，同样主证书也不能成为客户端。

These are fully supported by Puppet Labs, which means:

如下情况意味着被Puppet Labs完全支持：

* Issues that arise in one of these three arrangements are considered **bugs,** and we'll fix them ASAP.
* 以上三种情况中任意一种出现的问题被认为是 **bug，**我们会尽快修复。
* Issues that arise in any _other_ external CA setup are considered **feature requests,** and we'll consider whether to expand our support.
* 任何_其他_外部证书颁发机构(CA)出现的问题被认为是 **特性需求，**我们会考虑是否扩充至我们的支持范围中。

These configurations are all-or-nothing rather than mix-and-match. When using an external CA, the built in Puppet CA service **must** be disabled and cannot be used to issue SSL certificates.

以上配置是要么全有要么全无的配置而不是混合或匹配的配置。当使用外部证书颁发机构(CA)时，Puppet CA服务的构建 **必须** 被禁止并且不能被用以颁发SSL证书。

Additionally, Puppet cannot automatically distribute certificates in these configurations --- you must have your own complete system for issuing and distributing certificates.

另外，Puppet在以上配置中不能自动分发证书 --- 你必须拥有自有的完整系统用以颁发并且分发证书。
**`>>>这句存在疑问`**

General Notes and Requirements
-----

通用事项及需求
-----

### Rack Webserver is Required
### 需要支持Rack的web服务器

The puppet master must be running inside of a Rack-enabled web server, not the built-in Webrick server.

puppet主控必须运行在支持Rack的web服务器中，不能使用内置的Webrick服务器。

In practice, this means Apache or Nginx.  We fully support any web server that can:

在实际中，这意味着使用Apache或Nginx。我们完全支持任何拥有如下功能的web服务器：

* Terminate SSL
* Verify the authenticity of a client SSL certificate
* Set two request headers for:
    * Whether the client was verified
    * The client's distinguished name

* SSL调度
* 验证客户端SSL证书真伪
* 设置两种请求包头：
	* 客户端是否已验证
	* 客户端的识别名称

### PEM Encoding of Credentials is Mandatory

### 证书的PEM编码是强制性的

Puppet always expects its SSL credentials to be in `.pem` format.

Puppet一直期望它的SSL证书是 `.pem` 格式的。

### Normal Puppet Master Certificate Requirements Still Apply

### 常规的Puppet主控证书的需求仍然存在

Any puppet master certificate must contain the DNS name at which agent nodes will attempt to contact that master, either as the subject CN or as a Subject Alternative Name (DNS).

任何puppet主控证书必须包含DNS名，即代理节点联系主控节点的名称，无论是CN项目还是备用名称项目(DNS)。
**`>>>这句存在疑问`**

### Format of X-Client-DN Request Header

### 请求包头的X-Client-DN格式

Rack web servers must set a client request header, which the puppet master will check based on the [`ssl_client_header` setting](/references/latest/configuration.html#sslclientheader).

支持Rack的web服务器必须设置客户端的请求包头，puppet将以[`ssl_client_header`设置](/references/latest/configuration.html#sslclientheader)检查此包头。

This header should conform to the following specifications:

包头需要符合以下条件：

* The value of the client certificate DN should be in [RFC-2253](http://www.ietf.org/rfc/rfc2253.txt) format. The format of the `SSL_CLIENT_S_DN` environment variable (set by Apache ≥ 2.2's `mod_ssl`) is fully supported.
* Alternatively, the value of this request header may be in "OpenSSL" format.

* 客户端的证书DN值需要遵从 [RFC-2253](http://www.ietf.org/rfc/rfc2253.txt) 格式。环境变量`SSL_CLIENT_S_DN`(由Apache2.2及以上版本的 `mod_ssl` 设置)的格式完全被支持。

### Revocation

### 吊销

Certificate revocation list (CRL) checking works in all three supported
configurations, so long as the CRL file is distributed to the agents and masters
using an "out of band" process.  Puppet won't automatically update the CRL on any
of the components in the system.

证书吊销列表(CRL)的检查工作在三种支持配置中都可以使用，只要CRL文件使用“带外”过程向代理及主控分发即可。Puppet将不会在系统的任何组件中自动更新CRL。

#### If Unused:

#### 如果未使用CRL：

If revocation lists are **not** being used by the external CA, you must disable CRL checking on the agent.
Set `certificate_revocation = false` in the
`[agent]` section of [puppet.conf][conf] on **every agent node.**

如果吊销列表 **不** 为外部CA使用，你必须在代理上禁止CRL检查。
在 **每个代理节点** [puppet.conf][conf]的`[agent]`中，由 `certificate_revocation = false` 进行设置。

(If it's not set to false and the agent doesn't already have a CRL file, it will try to download one from the master. This will fail, because the master must have the CA service disabled.)

(如果代理不存在CRL文件并且已设置为false，它将会试图从主控下载一个。这将导致失败，因为主控必须禁止CA服务。)

#### If Used:

#### 如果使用CRL：

If revocation lists **are** being used by the external CA, then the CRL file must
be manually distributed to **every agent node** as a PEM encoded bundle.  Puppet will not automatically distribute this file.

如果吊销列表 **是** 为外部CA使用，CRL文件必须以PEM编码格式手动分发至 **每个代理节点**。 Puppet将不会自动分发这个文件。 

To determine where to put the CRL file, run `puppet agent --configprint hostcrl`.

可以运行 `puppet agent --configprint hostcrl` 以确定CRL文件的放置位置。

Option 1: Single CA
-----

第一种情形：独立CA
-----

A single CA is the default configuration of Puppet when the internal CA is
being used.  A single, externally issued CA may also be used in a similar
manner.

当内部CA启用时，puppet的默认配置为独立CA。由外部颁发证书的CA将采用类似的方式。


                   +------------------------+
                   |                        |
                   |  Root self-signed CA   |
                   |                        |
                   +------+----------+------+
                          |          |
               +----------+          +------------+
               |                                  |
               v                                  v
      +-----------------+                +----------------+
      |                 |                |                |
      | Master SSL Cert |                | Agent SSL Cert |
      |                 |                |                |
      +-----------------+                +----------------+

                   +------------------------+
                   |                        |
                   |        自签名根CA      |
                   |                        |
                   +------+----------+------+
                          |          |
               +----------+          +------------+
               |                                  |
               v                                  v
      +-----------------+                +----------------+
      |                 |                |                |
      |   主控SSL认证   |                |   代理SSL认证  |
      |                 |                |                |
      +-----------------+                +----------------+


### Puppet Master

### Puppet主控配置

{% capture master_basic %}
Configure the puppet master in four steps:

配置puppet主控分为四步：

1. Disable the internal CA service
2. Ensure that the certname will never change
3. Put certificates/keys in place on disk
4. Configure the web server

1. 禁止内部CA服务
2. 确保certname不会被修改
3. 将证书/密钥存放于磁盘中
4. 配置web服务器

On the master, in [`puppet.conf`][conf], make sure the following settings are configured:

在主控上，[`puppet.conf`][conf]中，确保配置了以下设置：

    [master]
    ca = false
    certname = <some static string, e.g. 'puppetmaster'>

    [master]
    ca = false
    certname = <一些静态字串，比如'puppetmaster'>

* The internal CA service must be disabled using `ca = false`.
* The certname must be set to a static value. This can still be the machine's FQDN, but you must not leave the setting blank. (A static certname will keep Puppet from getting confused if the machine's hostname ever changes.)

* 内部CA服务必须以 `ca = false` 禁止运行。
* certname必须设置为静态字串。可以同设备的FQDN，但是不能留空。(一个静态的certname将使得puppet不会受到设备hostname修改的影响。)

Once this configuration is set, put the external credentials into the correct filesystem locations.  You can run the following commands to find the appropriate locations:

一旦这项配置被配置，存放外部证书到正确的文件系统位置。你可以运行以下命令找寻适当的位置：

Credential                         | File location
-----------------------------------|-------------------------------------------
Master SSL certificate             | `puppet master --configprint hostcert`
Master SSL certificate private key | `puppet master --configprint hostprivkey`
Root CA certificate                | `puppet master --configprint localcacert`

证书                               | 文件位置
-----------------------------------|-------------------------------------------
主控SSL证书                        | `puppet master --configprint hostcert`
主控SSL证书私钥                    | `puppet master --configprint hostprivkey`
根CA证书                           | `puppet master --configprint localcacert`
{% endcapture %}

{{ master_basic }}

With these files in place, the web server should be configured to:

这些文件存放就绪后，配置web服务器如下：

* Use the root CA certificate, the master's certificate, and the master's key
* Set the verification header (as specified in the master's [`ssl_client_verify_header` setting][verify_header])
* Set the client DN header (as specified in the master's [`ssl_client_header` setting][client_header])

* 使用根CA证书，主控的证书，和主控的密钥
* 设置验证包头(在主控的[`ssl_client_verify_header`设置][verify_header]中配置)
* 设置客户端DN包头(在主控的[`ssl_client_header`设置][client_header]中配置)

An example of this configuration for Apache:

以下为一份Apache的配置样本：

    Listen 8140
    <VirtualHost *:8140>
        SSLEngine on
        SSLProtocol ALL -SSLv2
        SSLCipherSuite ALL:!ADH:RC4+RSA:+HIGH:+MEDIUM:-LOW:-SSLv2:-EXP

        # Replace with the value of `puppet master --configprint hostcert`
        SSLCertificateFile "/path/to/master.pem"
        # Replace with the value of `puppet master --configprint hostprivkey`
        SSLCertificateKeyFile "/path/to/master.key"

        # Replace with the value of `puppet master --configprint localcacert`
        SSLCACertificateFile "/path/to/ca.pem"

        SSLVerifyClient optional
        SSLVerifyDepth 1
        SSLOptions +StdEnvVars
        RequestHeader set X-SSL-Subject %{SSL_CLIENT_S_DN}e
        RequestHeader set X-Client-DN %{SSL_CLIENT_S_DN}e
        RequestHeader set X-Client-Verify %{SSL_CLIENT_VERIFY}e

        DocumentRoot "/etc/puppet/public"

        PassengerRoot /usr/share/gems/gems/passenger-3.0.17
        PassengerRuby /usr/bin/ruby

        RackAutoDetect On
        RackBaseURI /
    </VirtualHost>

{% capture master_config_ru %}
The `config.ru` file for rack has no special configuration when using an
external CA.  Please follow the standard rack documentation for using Puppet
with rack.  The following example will work with Puppet 3.2.

当使用外部CA时，rack的`config.ru`文件没有特殊配置。请遵循标准rack文档以配置Puppet使用rack。以下为一个运行在Puppet 3.2的例子。

    {% highlight ruby %}
    $0 = "master"
    ARGV << "--rack"
    ARGV << "--confdir=/etc/puppet"
    ARGV << "--vardir=/var/lib/puppet"
    require 'puppet/util/command_line'
    run Puppet::Util::CommandLine.new.execute
{% endhighlight %}
{% endcapture %}

{{ master_config_ru }}

### Puppet Agent

### Puppet代理配置

You don't need to change any settings.

你不需要修改任何设置。

Put the external credentials into the correct filesystem locations.  You can run the following commands to find the appropriate locations:

将外部证书存放至正确的文件系统位置。你可以运行以下命令找寻适当的位置：

Credential                        | File location
----------------------------------|-----------------------------------------
Agent SSL certificate             | `puppet agent --configprint hostcert`
Agent SSL certificate private key | `puppet agent --configprint hostprivkey`
Root CA certificate               | `puppet agent --configprint localcacert`

证书                              | 文件位置
----------------------------------|-----------------------------------------
代理SSL证书                       | `puppet agent --configprint hostcert`
代理SSL证书私钥                   | `puppet agent --configprint hostprivkey`
根CA证书                          | `puppet agent --configprint localcacert`

Option 2: Single Intermediate CA
-----

第二种情形：独立中间证书颁发机构
-----

The single intermediate CA configuration builds from the single self-signed CA
configuration and introduces one additional intermediate CA.

独立中间证书颁发机构的配置来自于独立自签名CA配置，它引入了一个额外的中间人CA。


                   +------------------------+
                   |                        |
                   |  self-signed Root CA   |
                   |                        |
                   +-----------+------------+
                               |
                               |
                               v
                   +------------------------+
                   |                        |
                   |    Intermediate CA     |
                   |                        |
                   +------+----------+------+
                          |          |
               +----------+          +------------+
               |                                  |
               v                                  v
      +-----------------+                +----------------+
      |                 |                |                |
      | Master SSL Cert |                | Agent SSL Cert |
      |                 |                |                |
      +-----------------+                +----------------+

                   +------------------------+
                   |                        |
                   |      自签名根CA        |
                   |                        |
                   +-----------+------------+
                               |
                               |
                               v
                   +------------------------+
                   |                        |
                   |  独立中间证书颁发机构  |
                   |                        |
                   +------+----------+------+
                          |          |
               +----------+          +------------+
               |                                  |
               v                                  v
      +-----------------+                +----------------+
      |                 |                |                |
      |   主控SSL认证   |                |   代理SSL认证  |
      |                 |                |                |
      +-----------------+                +----------------+


The Root CA does not issue SSL certificates in this configuration.  The
intermediate CA issues SSL certificates for clients and servers alike.

在此配置中根CA并不颁发SSL证书。中间证书颁发机构为客户端及服务器颁发SSL证书。

### Puppet Master

### puppet主控配置

{{ master_basic }}

You must also create a **CA bundle** for the web server. Append **the two CA certificates** together; the Root CA certificate must be located after the intermediate CA certificate within the file.

你必须为web服务器创建一个 **CA集**。同时附加至 **两个CA证书**；此文件中，根CA证书必须位于中间证书颁发机构证书之后。

    $ cat intermediate_ca.pem root_ca.pem > ca_bundle.pem

Put this file somewhere predictable. Puppet doesn't use it directly, but it can live alongside Puppet's copies of the certificates.

将此文件存放于可预测的某处。Puppet不会直接使用它，但它可以与puppet的证书副本共存与某处。
**`>>>这句存在疑问`**

With these files in place, the web server should be configured to:

文件存放以后，web服务器应当配置如下：

* Use the intermediate+Root CA bundle, the master's certificate, and the master's key
* Set the verification header (as specified in the master's [`ssl_client_verify_header` setting][verify_header])
* Set the client DN header (as specified in the master's [`ssl_client_header` setting][client_header])

* 使用中间+根的CA集，主控的证书，主控的密钥
* 设置验证包头(在主控的[`ssl_client_verify_header`设置][verify_header]中配置)
* 设置客户端DN包头(在主控的[`ssl_client_header`设置][client_header]中配置)

An example of this configuration for Apache:

以下为一份Apache的配置样本：

    Listen 8140
    <VirtualHost *:8140>
        SSLEngine on
        SSLProtocol ALL -SSLv2
        SSLCipherSuite ALL:!ADH:RC4+RSA:+HIGH:+MEDIUM:-LOW:-SSLv2:-EXP

        # Replace with the value of `puppet master --configprint hostcert`
        SSLCertificateFile "/path/to/master.pem"
        # Replace with the value of `puppet master --configprint hostprivkey`
        SSLCertificateKeyFile "/path/to/master.key"

        # Replace with the value of `puppet master --configprint localcacert`
        SSLCACertificateFile "/path/to/ca_bundle.pem"
        SSLCertificateChainFile "/path/to/ca_bundle.pem"

        # Allow only clients with a SSL certificate issued by the intermediate CA with
        # common name "Puppet CA"  Replace "Puppet CA" with the CN of your
        # intermediate CA certificate.
        SSLRequire %{SSL_CLIENT_I_DN_CN} eq "Puppet CA"

        SSLVerifyClient optional
        SSLVerifyDepth 2
        SSLOptions +StdEnvVars
        RequestHeader set X-SSL-Subject %{SSL_CLIENT_S_DN}e
        RequestHeader set X-Client-DN %{SSL_CLIENT_S_DN}e
        RequestHeader set X-Client-Verify %{SSL_CLIENT_VERIFY}e

        DocumentRoot "/etc/puppet/public"

        PassengerRoot /usr/share/gems/gems/passenger-3.0.17
        PassengerRuby /usr/bin/ruby

        RackAutoDetect On
        RackBaseURI /
    </VirtualHost>


{{ master_config_ru }}

### Puppet Agent

### Puppet代理配置

With an intermediate CA, puppet agent needs a modified value for [the `ssl_client_ca_auth` setting][ca_auth] in its puppet.conf:

使用中间CA的puppet代理需要在它的puppet.conf中修改[`ssl_client_ca_auth`设置][ca_auth]的值：

    [agent]
    ssl_client_ca_auth = $certdir/issuer.pem

The value should point to somewhere in the `$certdir`.

这个值需要指向 `$certdir` 中的某处。

Put the external credentials into the correct filesystem locations.  You can run the following commands to find the appropriate locations:

将外部证书存放至正确的文件系统位置。你可以运行以下命令找寻适当的位置：

Credential                        | File location
----------------------------------|------------------------------------------------
Agent SSL certificate             | `puppet agent --configprint hostcert`
Agent SSL certificate private key | `puppet agent --configprint hostprivkey`
Root CA certificate               | `puppet agent --configprint localcacert`
Intermediate CA certificate       | `puppet agent --configprint ssl_client_ca_auth`

证书                              | 文件位置
----------------------------------|-----------------------------------------
代理SSL证书                       | `puppet agent --configprint hostcert`
代理SSL证书私钥                   | `puppet agent --configprint hostprivkey`
根CA证书                          | `puppet agent --configprint localcacert`
中间CA证书                        | `puppet agent --configprint localcacert`

Option 3: Two Intermediate CAs Issued by One Root CA
-----

第三种情形：两个中间CA，证书颁发自同一个根CA


This configuration uses different CAs to issue puppet master certificates and puppet agent
certificates.  This makes them easily distinguishable, and prevents any agent certificate from being
usable for a puppet master.

这种配置使用不同的CA向puppet主控及代理颁发证书。可以使其易于区分，并且可以防止任何puppet代理的证书应用到puppet主控上。



                   +------------------------+
                   |                        |
                   |  Root self-signed CA   |
                   |                        |
                   +------+----------+------+
                          |          |
               +----------+          +------------+
               |                                  |
               v                                  v
      +-----------------+                +----------------+
      |                 |                |                |
      |    Master CA    |                |    Agent CA    |
      |                 |                |                |
      +--------+--------+                +--------+-------+
               |                                  |
               |                                  |
               v                                  v
      +-----------------+                +----------------+
      |                 |                |                |
      | Master SSL Cert |                | Agent SSL Cert |
      |                 |                |                |
      +-----------------+                +----------------+

                   +------------------------+
                   |                        |
                   |       自签名根CA       |
                   |                        |
                   +------+----------+------+
                          |          |
               +----------+          +------------+
               |                                  |
               v                                  v
      +-----------------+                +----------------+
      |                 |                |                |
      |     主控CA      |                |     代理CA     |
      |                 |                |                |
      +--------+--------+                +--------+-------+
               |                                  |
               |                                  |
               v                                  v
      +-----------------+                +----------------+
      |                 |                |                |
      |   主控SSL认证   |                |   代理SSL认证  |
      |                 |                |                |
      +-----------------+                +----------------+

In this configuration puppet agents are configured to only authenticate peer
certificates issued by the Master CA.  Puppet masters are configured to only
authenticate peer certificates issued by the Agent CA.

在这种配置中puppet代理被配置为仅认证由同级主控CA颁发的证书。puppet主控配置为仅认证由同级代理CA颁发的证书。

> **Note:** If you're using this configuration, you can't use the ActiveRecord inventory service backend with multiple puppet master servers. Use [PuppetDB][] for the inventory service instead.

> **注意：** 如果你不使用这种配置，你不能在拥有多puppet主控服务器的后端使用ActiveRecord inventory service。使用[PuppetDB][]作为inventory service的替代。

### Puppet Master

### puppet主控配置

{{ master_basic }}

You must also create a **CA bundle** for the web server. Append the **Agent CA certificate and Root CA certificate** together; the Root CA certificate must be located after the Agent CA certificate within the file.

你必须同样为web服务器创建一个 **CA集**。同时附加至 **代理CA证书及根CA证书**；此文件中，根CA证书必须位于代理CA证书之后。

    $ cat agent_ca.pem root_ca.pem > ca_bundle_for_master.pem

Put this file somewhere predictable. Puppet doesn't use it directly, but it can live alongside Puppet's copies of the certificates.

将此文件存放于可预测的某处。Puppet不会直接使用它，但它可以与puppet的证书副本共存与某处。
**`>>>这句存在疑问`**

With these files in place, the web server should be configured to:

文件存放以后，web服务器应当配置如下：

* Use the Agent+Root CA bundle, the master's certificate, and the master's key
* Set the verification header (as specified in the master's [`ssl_client_verify_header` setting][verify_header])
* Set the client DN header (as specified in the master's [`ssl_client_header` setting][client_header])

* 使用代理+根的CA集，主控的证书，主控的密钥
* 设置验证包头(在主控的[`ssl_client_verify_header`设置][verify_header]中配置)
* 设置客户端DN包头(在主控的[`ssl_client_header`设置][client_header]中配置)

An example of this configuration for Apache:

以下为一份Apache的配置样本：

    Listen 8140
    <VirtualHost *:8140>
        SSLEngine on
        SSLProtocol ALL -SSLv2
        SSLCipherSuite ALL:!ADH:RC4+RSA:+HIGH:+MEDIUM:-LOW:-SSLv2:-EXP

        # Replace with the value of `puppet master --configprint hostcert`
        SSLCertificateFile "/path/to/master.pem"
        # Replace with the value of `puppet master --configprint hostprivkey`
        SSLCertificateKeyFile "/path/to/master.key"

        # Replace with the value of `puppet master --configprint localcacert`
        SSLCACertificateFile "/path/to/ca_bundle_for_master.pem"
        SSLCertificateChainFile "/path/to/ca_bundle_for_master.pem"

        # Allow only clients with a SSL certificate issued by the intermediate CA with
        # common name "Puppet Agent CA"  Replace "Puppet Agent CA" with the CN of your
        # Agent CA certificate.
        SSLRequire %{SSL_CLIENT_I_DN_CN} eq "Puppet Agent CA"

        SSLVerifyClient optional
        SSLVerifyDepth 2
        SSLOptions +StdEnvVars
        RequestHeader set X-SSL-Subject %{SSL_CLIENT_S_DN}e
        RequestHeader set X-Client-DN %{SSL_CLIENT_S_DN}e
        RequestHeader set X-Client-Verify %{SSL_CLIENT_VERIFY}e

        DocumentRoot "/etc/puppet/public"

        PassengerRoot /usr/share/gems/gems/passenger-3.0.17
        PassengerRuby /usr/bin/ruby

        RackAutoDetect On
        RackBaseURI /
    </VirtualHost>


{{ master_config_ru }}

### Puppet Agent

### Puppet代理配置

With split CAs, puppet agent needs a modified value for [the `ssl_client_ca_auth` setting][ca_auth] in its puppet.conf:

使用分离CA的puppet代理需要在它的puppet.conf中修改[`ssl_client_ca_auth`设置][ca_auth]的值：

    [agent]
    ssl_client_ca_auth = $certdir/ca_master.pem

The value should point to somewhere in the `$certdir`.

这个值需要指向 `$certdir` 中的某处。

Put the external credentials into the correct filesystem locations.  You can run the following commands to find the appropriate locations:

将外部证书存放至正确的文件系统位置。你可以运行以下命令找寻适当的位置：

Credential                        | File location
----------------------------------|------------------------------------------------
Agent SSL certificate             | `puppet agent --configprint hostcert`
Agent SSL certificate private key | `puppet agent --configprint hostprivkey`
Root CA certificate               | `puppet agent --configprint localcacert`
Master CA certificate             | `puppet agent --configprint ssl_client_ca_auth`

证书                              | 文件位置
----------------------------------|-----------------------------------------
代理SSL证书                       | `puppet agent --configprint hostcert`
代理SSL证书私钥                   | `puppet agent --configprint hostprivkey`
根CA证书                          | `puppet agent --configprint localcacert`
主控CA证书                        | `puppet agent --configprint localcacert`
