---
title: 用OpenWrt实现同Wifi双密码
date: 2026-03-28 12:07:04
tags: [Misc, OpenWrt]
---

## 0xFF 引言
最近入手了个路由器，用来接校园网的宽带（神秘校园网只允许一个设备同时接入..）
正当我在想Wifi名的时候，突然想整个活，于是便设置成了`Ciallo～ (∠・ω )⌒☆`，
密码也是喜闻乐见的`0d000721`。

但设置好后突然感觉不对：这不是相当于公开了我花钱办的宽带了吗🤔于是便想改

但最后思来想去还是不想放弃这个Wifi名称，于是便设法实现了使用两个不同密码
连接会有不同响应的Wifi。具体来说，同一个名为`Ciallo～ (∠・ω )⌒☆`的网络，
使用`0d000721`连接就只会弹一个带图片的网址，
而使用另外一个密码连接才能正常上网。


<div style="display: flex; justify-content: center; text-align: center;">
  <figure>
    <img src="https://zcraftasserts-1302810751.cos.ap-shanghai.myqcloud.com/blog/_posts/openwrt-one-wifi-diff-pass/resnrm.png" width="300px"/>
  </figure>
  
  <figure>
    <img src="https://zcraftasserts-1302810751.cos.ap-shanghai.myqcloud.com/blog/_posts/openwrt-one-wifi-diff-pass/rescia.jpg" width="300px"/>
  </figure>
</div>

## 0x00 准备工作

想要实现上述效果，需要：

- 一台已刷OpenWrt的路由器（关于路由器的选购与刷机这里不赘述）
- 基础Shell操作知识
- 正常的网络连接

注意：
- 本教程使用的OpenWrt版本为`25.XX`，不建议在更早的版本上使用本教程，
，因为不同版本间的差异较大，可能会无法使用或造成故障。
- 搞机有风险，操作需谨慎🥹

## 0x01 连接网络

如果你的路由器已经连接到网络，可以跳过本节。

打开OpenWrt的管理页面，选择`网络>无线`。这里便是管理你设备的无线网络的地方。

这里我们使用客户端模式连接到网络并做中继。
点击任意`radio`的`扫描`按钮打开网络扫描，在弹出的框中
选择要连接的网络点击`加入网络`，之后在`WPA 密钥`中输入
密码，点击提交，再点击保存，最后点击最下面的`保存并应用`，
等待设置更新。

> 这里不同的`radio`可能分别代表的是2.4GHz和5GHz的天线，具体取决于你自己的设备。
> 图中以`802.11ax/b/g/n`结尾的为2.4GHz天线，以`802.11ac/ax/n`结尾的为5GHz天线。

![](https://zcraftasserts-1302810751.cos.ap-shanghai.myqcloud.com/blog/_posts/openwrt-one-wifi-diff-pass/connectInet.png)

## 0x02 创建Wifi

接着我们创建需要使用不同密码连接的Wifi。

点击任意`radio`的`添加`按钮打开网络扫描，在弹出的框中翻到最下面，
在`ESSID`框中输入要使用的Wifi名称。

![](https://zcraftasserts-1302810751.cos.ap-shanghai.myqcloud.com/blog/_posts/openwrt-one-wifi-diff-pass/create-ssid.png)

切换到`无线安全`，将`加密`改为`WPA2-PSK`，之后在`密钥`中输入**真实**的密码（之后能够正常联网的密码）

![](https://zcraftasserts-1302810751.cos.ap-shanghai.myqcloud.com/blog/_posts/openwrt-one-wifi-diff-pass/create-pass.png)

点击`保存`，最后点击最下面的`保存并应用`，等待设置更新。

## 0x03 升级`dnsmasq-full`

首先由于我使用的是OpenNDS来做弹出的页面，所以需要将`dnsmasq`包升级为完整版。

在Shell中输入以下命令：

```shell
apk update && apk add dnsmasq-full
```

看到形如`OK: XXX MiB in XXX packages`的结果即可。

## 0x04 配置Multi-PSK

使用Shell编辑`/etc/config/wireless`

> 关于Shell中的文本编辑器Vim的操作，请详见[此文章](https://www.runoob.com/linux/linux-vim.html)

以下是一个示例：

```
config wifi-device 'radio0'
    ......

config wifi-iface 'default_radio0'
    ......

config wifi-device 'radio1'
    ......

config wifi-iface 'wifinet3'
    option device 'radio1'
    option network 'lan'
    option mode 'ap'
    option ssid 'hyw'
    option encryption 'psk2'
    option key 'ThisIsTheTruePassword'

config wifi-iface 'wifinet2'
    option device 'radio1'
    option network 'wwan'
    option ssid 'ExampleWifi'
    ......
```

可以看到，在`config wifi-iface 'default_radio1'`配置块中有我们刚刚创建的Wifi，
在`config wifi-iface 'wifinet2'`配置块中有我们刚刚连接的Wifi。

接下来对配置文件做以下更改：

### 1.启用MultiPSK
在创建的Wifi的配置块中加入`option multi_psk '1'`
之后你的配置应该像这样：
```
config wifi-iface 'wifinet3'
    option device 'radio1'
    option network 'lan'
    option mode 'ap'
    option ssid 'hyw'
    option encryption 'psk2'
    option key 'ThisIsTheTruePassword'
    option multi_psk '1'
```
### 2.添加`wifi-vlan`块

在文件底部添加以下配置：
```
config wifi-vlan
  option name 'g_v'
  option network 'guest'
  option vid '10'
  option iface 'wifinet3'
```
注意：
- 这里的`wifinet3`为我们创建的wifi的`wifi-iface`的名字，见上面的示例配置。
- 这里的`name`可以任取但是不能太长，具体限制在不同设备上
可能不同，但建议只使用3~4个字符。
- `vid`的`10`和`network`的`guest`均可自定义，不过之后需要保证配置能够对的上。

### 3.添加`wifi-station`块

继续在文件底部添加以下配置：

```
config wifi-station
    option key '0d000721'
    option vid '10'
    option iface 'wifinet3'
```

注意：
- 这里的`key`即为**假密码**，即无法正常上网的密码。
- 这里的`vid`和`iface`要和上面匹配。

编辑完成，保存文件进行下一步。

## 0x05 添加设备与接口

打开OpenWrt管理面板，进入`网络>接口`，切换到`设备`页面，点击最下面的`添加设备配置`。

在弹出的窗口中，将`设备类型`选为`网桥设备`,
`设备名称`设为`br-guest`（可以自己改），选中`允许启动空网桥`，然后点击`保存`。

![](https://zcraftasserts-1302810751.cos.ap-shanghai.myqcloud.com/blog/_posts/openwrt-one-wifi-diff-pass/create-dev.png)

接着切换回`接口`页面，点击下面的`添加新接口`，
将名称设置为`guest`（注意和上面的匹配），将`协议`设置为`静态地址`，将设备选为`br-guest`（和上面创建的匹配）,然后点击`创建接口`。

![](https://zcraftasserts-1302810751.cos.ap-shanghai.myqcloud.com/blog/_posts/openwrt-one-wifi-diff-pass/create-iface.png)

接着在弹出的窗口中，将`IPv4 地址`设置为`192.168.72.1`（可以任意设置，但要和你正常使用的主网络不同），将`IPv4 子网掩码`设置为`255.255.255.0`。

![](https://zcraftasserts-1302810751.cos.ap-shanghai.myqcloud.com/blog/_posts/openwrt-one-wifi-diff-pass/conf-iface.png)

然后点击上面的`防火墙设置`，点击`创建/分配防火墙区域`，在最下面的`自定义`框中输入`guest_zone`（可以自己起名）然后按回车。

![](https://zcraftasserts-1302810751.cos.ap-shanghai.myqcloud.com/blog/_posts/openwrt-one-wifi-diff-pass/conf-fwall.png)


最后点击上面的`DHCP 服务器`，然后点击`配置 DHCP 服务器`按钮，不用变更配置。

最后点击保存关闭配置窗口，点击最下面的`保存并应用`，等待设置更新。

## 0x06 安装`OpenNDS`

`OpenNDS`的作用本来一般是提供上网认证界面，但这里我们使用它来做弹出图片的效果。

打开Shell，输入以下命令：

```shell
apk update && apk add opennds
```

之后编辑`/etc/config/opennds`，找到`option gatewayinterface`行
（我设备上是第308行，可以使用查找功能查找），
将其取消注释并把后面的值换成上面配置的设备的名称。

更换之后配置应该如下：

```
...

# GateWayInterface
# Default br-lan
# Use this option to set the device opennds will bind to.
# The value may be an interface section in /etc/config/network or adevice name such as br-lan.
# The selected interface must be allocated an IPv4 address.
# In OpenWrt this is normally br-lan, in generic Linux it might besomething else.
#
option gatewayinterface 'br-guest'
##################################################################

...
```

保存文件，进行下一步。

## 0x07 配置弹出页面

编辑`/usr/lib/opennds/theme_click-to-continue-basic.sh`，
将里面的内容全部删除，替换成下面的：

```shell
#!/bin/sh
cat /etc/opennds/htdocs/splash.html
exit 0
```

之后保存文件。

然后编辑`/etc/opennds/htdocs/splash.html`（即上面配置的文件），将原有内容（如果有）全部删除，替换成你想要的内容。

在使用假密码登录时，OpenNDS会拦截链接并把`/etc/opennds/htdocs/splash.html`里面的HTML提供给登录者作为登录界面，所以这里可以自由发挥。

值得注意的是，由于使用假密码连接的时候没有网络连接，所以如果想要插入图片的话不能使用图床，因为无法访问。这里我为了方便直接使用了Base64硬编码图片到`splash.html`里。

另外，由于存在缓冲区大小限制，`splash.html`不能太长，所以如果要使用Base64添加图片的话需要对图片进行充分压缩。

以下是我的配置：

```html
<!DOCTYPE html>
<html>
<head>
    <title>Ciallo～ (∠・ω )⌒☆</title>
</head>
<body>
    <h1>Ciallo～ (∠・ω )⌒☆</h1><br>
    <img width="300px" id="ciallo" alt="ciallo" src="data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACgAAAAgCAIAAADvz6...(太长了省略)" />
</body>
</html>
```

保存文件，至此配置已经全部完成。

## 0x08 重启服务

使用以下命令重启服务：

```shell
/etc/init.d/network restart
wifi reload
/etc/init.d/dnsmasq restart
/etc/init.d/opennds restart
```

之后应该就可以看到创建的`Ciallo～ (∠・ω )⌒☆`Wifi了。

## 0x09 遇到问题？

由于本人之前也没有接触过OpenWrt，在尝试实现这个效果的过程中也是在摸索，
所以如果各位遇到了一些问题我可能无法进行有效的解答。不过这里有些我
遇到过的问题，供大家参考。

### 在配置完OpenNDS并重启后无法访问LuCI？

这一般是因为`OpenNDS`没有被正确配置（即`gatewayinterface`没有与前面
创建的设备匹配），导致它将所有网络连接全部拦截。

遇到这种情况时，使用Shell登录，并执行`/etc/init.d/opennds stop`
来停止`OpenNDS`，之后应该就能恢复LuCI访问。
为了解决问题，请参照[上面的步骤](#0x06-安装OpenNDS)正确配置之后，再运行`/etc/init.d/opennds start`
恢复`OpenNDS`，看看是否正常工作。

### 重启服务后找不到添加的Wifi？

这可能是因为你的`wifi-vlan`块中的`name`过长，
请参照[上面的步骤](#2-添加wifi-vlan块)将其改短后再执行`wifi reload`看看。

END.