Caddy Web 服务器的安全正向代理
该包注册了http.handlers.forward_proxy模块，它充当访问远程网络的 HTTPS 代理。

⚠️实验性的！
此模块为实验性模块。我们需要更多用户来测试此模块是否存在错误和弱点，然后我们才会推荐在受监控的网络或有主动审查的地区使用该模块。在个人安全、自由或隐私受到威胁的情况下，请勿依赖此代码。

您可以通过以下方式提供帮助：

安全部署此模块
试图打破它
为这个 repo 中的代码和测试做出贡献，使其变得更好
我们也在寻求有经验的维护人员，他们对这些技术有经验并且有兴趣继续开发。

预计会发生重大变化。

特征
HTTP/1.1、HTTP/2 和 HTTP/3 支持
验证
访问控制列表
可选探头电阻
PAC 文件
介绍
此 Caddy 模块允许您将 Web 服务器用作代理服务器，可由众多 HTTP 客户端（例如操作系统、Web 浏览器、移动设备和应用程序）配置。但是，每个客户端的功能集差异很大，其正确性和安全性保证也各不相同。您必须了解每个客户端各自的弱点或缺点。

快速启动
首先，您必须知道如何使用 Caddy。

用这个插件构建 Caddy。你可以从Caddy 的下载页面添加它，也可以用xcaddy自己构建它：

$ xcaddy build --with github.com/caddyserver/forwardproxy
大多数人更喜欢使用Caddyfile进行配置。您可以像这样建立一个简单、开放的未经身份验证的正向代理：

example.com
route {
	# UNAUTHENTICATED! USE ONLY FOR TESTING
	forward_proxy
}
（显然，替换example.com为指向您机器的域名。）

由于forward_proxy它不是标准指令，因此它相对于其他处理程序指令的顺序尚未定义，因此我们将其放在块内route。您也可以这样做：

{
	order forward_proxy before file_server
}
example.com
# UNAUTHENTICATED! USE ONLY FOR TESTING
forward_proxy
全局定义其位置；那么您不需要route块。正确的顺序由您决定，并取决于您的配置。

该插件使Caddy可以充当正向代理，支持 HTTP/3、HTTP/2 和 HTTP/1.1 请求。HTTP/3 和 HTTP/2 通常会因多路复用而提高性能。

正向代理插件包括访问控制列表和身份验证等常用功能，以及一些有助于保护安全和隐私的独特功能。正向代理的默认配置符合现有的 HTTP 标准，但某些功能会强制插件表现出非标准但不破坏的行为以保护隐私。

探测阻力 — 此插件的标志性功能之一 — 试图隐藏您的 Web 服务器也是正向代理的事实，帮助代理保持低调。最终，forwardproxy 插件实现了一个简单的反向代理（upstream https://user:password@next-hop.com在 Caddyfile 中），以便用户在需要反向代理时（例如，构建代理链）可以利用它probe_resistance。反向代理实现将保持简单，如果您需要强大的反向代理，请查看 Caddy 的标准proxy指令。

有关功能及其用法的完整列表，请参阅 Caddyfile 语法：

Caddyfile 语法（服务器配置）
启用无需身份验证的正向代理的最简单方法是将forward_proxy指令包含在 Caddyfile 中。但是，这允许任何人将您的服务器用作代理，这可能不是理想的选择。

该forward_proxy指令没有默认顺序，必须在route指令内使用以明确指定其评估顺序。在 Caddyfile 中，地址必须以 开头，:443才能forward_proxy适用于所有来源的代理请求。

以下是所有属性的使用示例（请注意，语法可能会发生变化）：

:443, example.com
route {
	forward_proxy {
		basic_auth user1 0NtCL2JPJBgPPMmlPcJ
		basic_auth user2 密码
		ports     80 443
		hide_ip
		hide_via
		probe_resistance secret-link-kWWL9Q.com # alternatively you can use a real domain, such as caddyserver.com
		serve_pac        /secret-proxy.pac
		dial_timeout     30
		upstream         https://user:password@extra-upstream-hop.com
		acl {
			allow     *.caddyserver.com
			deny      192.168.1.1/32 192.168.0.0/16 *.prohibitedsite.com *.localhost
			allow     ::1/128 8.8.8.8 github.com *.github.io
			allow_file /path/to/whitelist.txt
			deny_file  /path/to/blacklist.txt
			allow     all
			deny      all # unreachable rule, remaining requests are matched by `allow all` above
		}
	}
	file_server
}
（方括号[ ]表示您应该替换的值；实际上不包括括号。）

安全
basic_auth [user] [password]
设置基本 HTTP 身份验证凭据。此属性可以重复多次。请注意，这与 Caddy 的内置basic_auth指令不同。在输入凭据之前，请务必检查请求凭据的站点的名称。

默认值：无需身份验证。

probe_resistance [secretlink.tld]
尝试隐藏网站是正向代理的事实。如果凭证不正确或缺失，代理将不再响应“407 需要代理身份验证”，并将尝试模仿通用 Caddy Web 服务器，就好像未启用正向代理一样。

basic_auth只有在设置了 的情况下，探测阻力才会起作用（并且有意义） 。要使用具有探测阻力的代理，请将您的basic_auth凭据提供给您的客户端配置。如果您的代理客户端（浏览器、操作系统、浏览器扩展等）允许您预先配置凭据，并预先发送凭据，则您不需要秘密链接。

如果您的代理客户端不主动发送凭证，您将不得不访问浏览器中的秘密链接来触发身份验证。确保指定的域名是可访问的、不包含大写字符、不以点开头等。只有此地址才会触发 407 响应，提示浏览器向用户请求凭证并在会话的剩余时间内缓存它们。

默认值：无探测电阻。

隐私
hide_ip
如果设置，forwardproxy 将不会将用户的 IP 添加到“Forwarded：”标头。

警告：您的浏览器中还存在其他您可能需要消除的侧信道，例如 WebRTC，请在此处查看如何禁用它。

默认：不隐藏；Forwarded: for="useraddress"将被发送出去。

hide_via
如果设置，forwardproxy 将不会添加 Via 标头，并阻止以简单方式检测代理的使用情况。

警告：还有其他侧通道可以确定这一点。

默认：不隐藏；以 形式的标头Via: 2.0 caddy将被发送出去。

访问控制
ports [integer] [integer]...
指定 forwardproxy 将为所有请求列入白名单的端口。其他端口将被禁止。

默认：无限制。

acl {  
  acl_directive  
  ...  
  acl_directive  
}
指定允许的目标 IP 网络、IP 地址和主机名的顺序和规则。每个转发代理请求中的主机名将解析为 IP 地址，caddy 将按顺序根据指令检查 IP 地址和主机名，直到指令与请求匹配。

acl_directive或许：

allow [ip or subnet or hostname] [ip or subnet or hostname]...
allow_file /path/to/whitelist.txt
deny [ip or subnet or hostname] [ip or subnet or hostname]...
deny_file /path/to/blacklist.txt
如果你不希望不匹配的请求受到默认策略的约束，你可以在 acl 规则中添加以下一项来指定对不匹配的请求采取的操作：

allow all
deny all
对于hostname，您可以指定*.为前缀以匹配域和子域。例如， *.caddyserver.com将匹配caddyserver.com、subdomain.caddyserver.com，但不匹配fakecaddyserver.com。请注意，在链中早期匹配的主机名规则将覆盖后面的 IP 规则，因此建议将 IP 规则放在首位，除非域是高度可信的并且应该覆盖 IP 规则。另请注意，通过直接指定 IP，可以轻松绕过基于域的黑名单。

对于allow_file/deny_file指令，语法是相同的，并且每个条目必须用换行符分隔。

此策略适用于除代理自身域和端口的请求之外的所有请求。不支持按主机/IP 将端口列入白名单/黑名单。

默认策略：

acl {  
	deny 10.0.0.0/8 127.0.0.0/8 172.16.0.0/12 192.168.0.0/16 ::1/128 fe80::/10  
	allow all  
}
默认拒绝规则旨在禁止访问本地主机和本地网络，并且将来可能会扩展。

超时
dial_timeout [integer]
设置与目标网站建立 TCP 连接的超时时间（以秒为单位）。影响所有请求。

默认值：20 秒。

其他
serve_pac [/path.pac]
生成（内存中）并在给定路径上提供代理自动配置文件。如果未提供路径，PAC 文件将在 处提供/proxy.pac。注意：如果启用了探测阻力，您的 PAC 文件也应在秘密位置提供；在可预测的路径上提供该文件可以轻松击败探测阻力。

默认值：Caddy 不会生成或提供任何 PAC 文件（您仍然可以像常规文件一样手动创建和提供 proxy.pac）。

upstream [https://username:password@upstreamproxy.site:443]
设置上游代理以通过它路由所有转发代理请求。此设置不会影响非转发代理请求或具有错误凭据的请求。上游与acl和ports子指令不兼容。

支持的远程主机方案：https。

支持的本地主机方案：socks5、http、https（忽略证书检查）。

默认值：无上游代理。

获取转发代理
下载预构建的二进制文件
二进制文件位于https://caddyserver.com/download
不要忘记添加http.forwardproxy插件。

从源代码构建
安装最新的 Golang 1.20 或更高版本并设置export GO111MODULE=on
go install github.com/caddyserver/forwardproxy/cmd/caddy@latest
构建的caddy二进制文件将存储在 $GOPATH/bin 中。
客户端配置
请注意，客户端支持存在很大差异，并且存在客户端可能不使用代理的极端情况（当代理应该或可以使用时）。您需要了解这些限制。

基本配置只是使用您的站点地址和端口（通常适用于所有协议 - HTTP、HTTPS 等）。如果启用了 .pac 文件，您还可以指定该文件。

阅读此博客文章，了解如何配置特定客户端。

执照
根据Apache 许可证授权

免责声明
使用风险自负。本软件按原样提供。使用本软件即表示您同意并声明，本软件的作者、维护者和贡献者对您可能遇到的任何风险、费用或问题不承担任何责任。考虑您的威胁模型并保持警惕。如果您发现缺陷或错误，请提交补丁并帮助改进！

此插件的初始版本由 Google 开发。这不是 Google 官方产品。
