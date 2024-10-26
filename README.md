# WireGuard
- 使用WG进行异地组网详细步骤教学与讨论。
- 基于系统：Windows 10 企业版

## 步骤目录
1. 软件准备
2. 组网
3. 共享设置

## 软件准备
1. 新系统请首先打开【控制面板】-【程序和功能】-【启用或关闭Windows功能】，选择启用【SMB 1.0/CIFS 文件共享支持】和【SMB直通】两个选项，并重启系统。
2. 访问[https://www.wireguard.com/](https://www.wireguard.com/)，下载并安装异地组网工具WG。

* 讨论：为什么要开启SMB，起因似乎是因为Windows10自某次更新开始因为安全性原因取消了默认基于SMB协议的文件共享。使得原先基于SMB的用户访问登录协议变得无法如同原先一样工作。笔者并不清楚重新下载安装这个功能是否意味着强制开启这个协议，所以这个操作似乎并无价值，总之由于打开也没什么问题，且能满足潜在的和linux进行文件互通的需求，所以一般是推荐打开。 *

## 组网
使用WG进行组网， ** 典型比较适用的场景比如 ** ：
1. 主从双机同处于大型内网域内（例如校园网，办公网等），可以通过IPv4内网网段实现互通。
2. 或者主机方具有IPv6公网地址可以实现某种程度上的不稳定直连。

在此场景下如果你希望多机进行私网域构建，使信息不暴露于公网或者公开内网，那么非常适合使用WG组网。

 ** 不适合的场景比如 ** :
1. 无论如何没有公网地址，需要使用转发。此情况下应有大量其他方案优于本方案。

### Wireguard组网细节
- 不是本文的重点，不在此处赘述
- 希望实现的结果，例如本文最终组成了以`192.168.1.1/8`为服务器，以`192.168.1.101/8`为客户端的私有网域，要求双方在关闭windows防火墙的情况下可以互相ping通。

### 组网需要进行的其他工作
1. 尝试使用动态域名解析实现WG客户端对服务端IP变化的实时追踪。需要购买一个域名，并在服务器端部署网卡监测软件。笔者使用的项目是[https://github.com/jeessy2/ddns-go](https://github.com/jeessy2/ddns-go)，各位可以自行选择自己喜欢的。
2. 对WG组成的网络进行属性设置，以实现公开网络和私有网络的区分。我们仅希望在私有网络上暴露文件共享服务，故应该将校园网等网络设置为公开网络，将WG组成的虚拟适配器设置为专用网络，具体操作如下：
```powershell
# 在windows网络的图形设置界面可以将wifi本身设置为公开网络
# 然后在powershell中运行如下命令

Get-NetConnectionProfile # 查询当前网卡区域

# 例如我们的校园网络叫University Free WIFI，而私有网卡叫WG_Client，则通过以下命令将虚拟适配器设置为专用网络。

Set-NetConnectionProfile -Name "WG_Client" -NetworkCategory Private # 或Public
```
## 共享设置
Windows 10 最新版进行基础文件共享需要进行以下几个方面的设置
1. 账户设置：启用共享账户
2. 高级共享设置：控制全局网络共享访问情况
3. 组策略设置：控制具体共享账号情况
4. 凭据管理：配置客户端登录凭证
5. 共享文件夹设置：配置具体共享文件夹情况


* 注：如无特殊标注，本章节内容需要服务端和客户端双向都进行配置 *

### 1. 账户设置
首先我们为网络共享专门配置一个权限为普通用户组的用户，这在一定程度上是进行任何涉网络的权限敏感操作的通用处理办法。
- 右键点击【我的电脑】，选择【管理】来打开计算机管理。选择【本地用户和组】-【用户】，右键空白处选择【新用户】来创建新用户。
- 例如此处创建的用户为`mynetuser`
- 您需配置一个用户名以及一组非弱密码（12位以上，包含大小写字母，数字，特殊符号）。
- 新建账户时选择【用户不能更改密码】，且【密码永不过期】。

### 2. 高级共享设置
此部分配置实现了在私有局域网中的文件共享，而不使共享暴露于公开网络
- 打开【我的电脑】，右键选择【网络】-【属性】。或者，在系统搜索框里输入【网络】，并打开【网络和共享中心】
- 打开【更改高级网络共享设置】
- 其中，在【专用网络】部分选择【启用网络发现】、【启用文件和打印机共享】。
- 在【来宾或公用】部分选择【关闭网络发现】、【关闭文件和打印机共享】。
- 在【所有网络】部分选择【启用共享以便可以访问网络的用户可以读取和写入公用文件夹中的文件】、【使用128位加密帮助保护文件共享连接（推荐）】、【有密码保护的共享】
- 点击保存更改

### 3. 组策略设置
此部分实现了私有网络下进行共享访问时，对访问来源的账号权限控制，以避免不小心连接到其他私有网络时暴露共享服务。
- 按下`winkey + R`并输入`gpedit.msc`打开组策略配置
- 依次打开【计算机配置】-【Windows设置】-【安全设置】-【本地策略】-【安全选项】，找到【网络访问：本地账户的共享和安全模型】，将其设置为【经典-对本地用户进行身份验证，不改变其本来身份】
- 同时将【网络访问：不允许SAM账户的匿名枚举】和【网络访问：不允许SAM账户共享的匿名枚举】都设置为【已启用】
- 依次打开【计算机配置】-【Windows设置】-【安全设置】-【本地策略】-【用户权限分配】
- 找到其中的【拒绝本地登录】，以及【拒绝从网络访问这台计算机】，向其中添加用户，确保`Guest`账号出现在这两项拒绝当中。这样可以禁止所有空密码访问的账户登录。
- 找到【从网络访问此计算机】，删除其中原有的`Everyone`, `Administrators`等所有选项，并添加唯一指定账户`mynetuser`
- 重启电脑
- 此时从其他主机访问此网络位置时得到的正常反馈为访问失败，例如访问`\\192.168.1.1`会出现如下提示：
```
请与这台服务器的管理员联系以查明你是否有访问权限。

登录失败: 未授予用户在此计算机上的请求登录类型。
```
此时说明已经对登录账号进行限制，无法进行匿名访问。

### 4. 凭据管理
- 在windows搜索框中输入【凭据管理器】或在控制面板中将其打开
- 找到【Windows凭据】-【添加Windows凭据】，输入`192.168.1.1`, `mynetuser`, `非弱密码`，点击【确定】
- 此后本机在访问`192.168.1.1`时会默认尝试使用此用户名密码登录。此时访问`\\192.168.1.1`，正常应不会出现上文所示的拦截，组策略设置的对应账户规则已经被实现。

### 5. 共享文件夹设置
- 右键选择您想共享的文件夹，打开【属性】，选择【共享】-【共享】。
- 添加刚刚创建的用户`mynetuser`并将权限设置为允许读取和写入，同时记得删除【Everyone】等通用性用户的权限。对于所有文件夹的`Administrators`账户则不需进行改动，再点击【确定】（如果未经3/4步骤组策略和登录凭据设置，则此处用户会以匿名身份登录，那么就必须开放Everyone权限才能够实现共享。反之，经过3/4步设置后，登录用户已经被严格限制为`mynetuser`，故可以抛开Everyone，对其实现精准权限控制。）
- 同理，选择【高级共享】，勾选【共享此文件夹】，点击【权限】，并对其中进行如同上一步一样的对`Everyone`的控制，和对`mynetuser`的权限开放。点击【确定】
- 设置全部完成，重启电脑。

至此，您应该能够实现仅限于私有网络的基于用户名/密码安全性的文件共享系统。


## 讨论

### 本文实现了什么？
- 本文实现了基于WG的异地组网+文件共享系统。您可以实现多机数据连通。如果您的组织网络不是特别差的话，一般能形成比较良好的办公、学习体验，而不需要背着数据到处跑了。

### 本方案安全性如何？
- 您进行了局限于异地虚拟内网的文件共享，并以密码对其进行安全保护。
- 由于服务未暴露于公网，通常来说，如果攻击者有意对您进行攻击，他需要物理接驳于您所在的域网络内进行扫描。而后对方还需要持有您的WG私钥才能建立连接。而建立连接后对方同样需要破译您的windows账户密码才可以正常访问数据，笔者认为安全性是足够的。
- 加入用户名/密码权限控制，可以保证当用户切换至其他私有网络时（例如您把笔记本带家庭WIFI后），新私有网络内的其他第三方仍需要持有密码才可以访问数据。

### 本文局限性
- 基于WG的组网目前只实现了【一主对多从】的组网模式，从机之间无法互相访问。即例如笔者有三台设备，工位的电脑、宿舍的电脑，两者可以实现完全数据互联。但如果使用第三设备手机访问，则只能连接二者中作为主机的电脑。如果希望多机组网可能需要依赖于Zerotier之类的工具，而不是WG。
- 暴露数据（并且基于一个易操作的图形界面）目前还有其他方案。除了windows自带的文件共享外，就笔者了解起码还有FTP、软件webdav的方案，且似乎同样能够实现账户安全性配置，还更简单。没有进行过详细调研。
