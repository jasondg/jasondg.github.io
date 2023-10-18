---
title: "Firefox Developer Edition 设置 desktop 快捷方式"
date: 2022-07-24T14:05:53+08:00
categories: ["log"]
tags: ["desktop", "firefox", "Xorg"]
draft: false
---

# desktop 快捷方式无法识别窗口

通过 bz 文件解压方式安装 Firefox Developer Edition 并手动创建 `firefox-developer.desktop` 后，desktop 文件无法正常使用，KDE plasma 5.25 下点击打开后窗口不能识别为同一 application，在 Task Manager 中会识别为新的窗口。

desktop 文件如下：

```
[Desktop Entry]
Name=Firefox Developer Edition
Comment=Web Browser
GenericName=Web Browser
Encoding=UTF-8
Exec=/opt/firefox/firefox %u
Icon=/opt/firefox/browser/chrome/icons/default/default128.png
Type=Application
StartupNotify=true
Terminal=false
Categories=Network;WebBrowser;GTK;
MimeType=text/html;application/xml;application/xhtml+xml;application/x-xpinstall;application/vnd.mozilla.xul+xml;
Actions=PrivateBrowsing;
Keywords=Firefox Developer Edition;

[Desktop Action PrivateBrowsing]
Exec=/opt/firefox/firefox --private-window %u
Name=New Private Browsing Window
Icon=/opt/firefox/browser/chrome/icons/default/default128.png
```

# 修改 StartupWMClass 解决窗口识别问题

在一个 Gist[^gh]中发现解决窗口识别问题需要设置`StartupWMClass`，但一些评论中的值无法生效。在 ask ubuntu 上有`StartupWMClass`的相关介绍[^au]，通过`xprop`/`xwininfo`可以获取窗口的 X11 属性，根据获取的`WM_Class`设置 desktop 文件即可解决窗口识别问题。

```
StartupWMClass=firefox-aurora
```

设置后打开窗口能够正确识别对应的 application。

# 后续问题

在 Firefox 默认使用 Wayland 后，`xprop`/`xwininfo` 将无法获取 `WM_Class` 值，而目前也没有官方文档标明了这些信息，如 Gist[^gh] 中评论所说，`WM_Class` 的值可能会变更，后续 Firefox 默认使用 Wayland 时可能需要另外的操作，也希望 Mozilla 之后能够增加 desktop 相关的说明。

[^gh]: https://gist.github.com/mahammad/e54e4be8938edf4d6c15
[^au]: https://askubuntu.com/questions/367396/what-does-the-startupwmclass-field-of-a-desktop-file-represent
