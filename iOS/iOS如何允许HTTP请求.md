### 背景

iOS9对应用网络安全做了升级，默认不支持 HTTP 请求，而要求使用 HTTPS 协议。

真正在开发中，一定会有很多 HTTP 协议的请求接口需要使用，所以下面说明如何允许 HTTP 请求。

### 使用

在项目的 Info.plist 文件中，增加 Property：
- key: NSAppTransportSecurity
- type: Dictionary
- NSAppTransportSecurity 中增加 NSAllowsArbitraryLoads，类型 BOOL，值为 YES

如需要增加部分域名，则结构如下：

    <key>NSAppTransportSecurity</key>
    <dict>
        <key>NSExceptionDomains</key>
        <dict>
            <key>xx.com</key>
            <dict>
                <key>NSIncludesSubdomains</key>
                <true/>
            </dict>
            <key>xxx.cn</key>
            <dict>
                <key>NSIncludesSubdomains</key>
                <true/>
            </dict>
        </dict>
    </dict>
