---
title: 建立MCP專案 with .Net
date: 2026-10-16 10:37:54
categories:
  - AI應用
tags:
  - AI
  - MCP
---
##### 用.Net建立一個測試用remote MCP Server專案
<!-- more -->
#
#
#
---
## 1. 建立Remote MCP專案

- **安裝範本：**
``` bash
dotnet new install Microsoft.McpServer.ProjectTemplates
```

- **建立MCP專案：**
如果要建立的是遠端MCP，要加上參數--transport remote。不然建立的MCP會是透過stdio進行資料傳輸的本地MCP。

``` bash
dotnet new mcpserver -n MyRemoteMcpServer --transport remote
cd MyRemoteMcpServer
dotnet run
```

- **確認MCP服務的port**
執行專案後，會看到正在監聽6527這個port
``` bash
dotnet run
```
{%asset_image check_port.jpg check_port %}
#
#
#

## 2. 測試Remote MCP
建立好測試的MCP專案後，藉著就是用claude desktop來測試看看了，結果出現下面的錯誤。<span style="color: red;">原因是 Claude 的自訂 connector 是由 Anthropic 的雲端伺服器去連你的 MCP，而不是從你的電腦發出請求。</span>所以 localhost 對它來說指向的是它自己，連不到你的機器。這是正常現象。
{%asset_image cant_use_localhost.jpg cant_use_localhost %}

- **暴露本機網址**
下面的做法可以把logcalhost曝露公開https網址：
1. Dev Tunnels（微軟，Visual Studio 內建，也有 CLI），會給你 https://xxxx.devtunnels.ms。
``` bash
winget install Microsoft.devtunnel
devtunnel host -p <port> --allow-anonymous
```

執行後，如果遇到下面錯誤要先登入devtunnel，會開瀏覽器讓你選帳號登入（Microsoft 或 GitHub 皆可）。
``` bash
devtunnel user login
```
取得對外的url
{% asset_image devtunnel_port.jpg devtunnel_port%}

建立Tunnel後你的終端機會被卡住，要開新的cmd視窗才可以進行後續操作，我們開新的CMD後，確認一下目前tunnel list
``` bash
devtunnel list
```
{% asset_image tunnel_list.jpg tunnel_list%}

再用剛剛透過devtunnel list取得的Tunnel ID去取得Tunnel的資訊
``` bash
devtunnel show <Tunnel ID>
```
{% asset_image tunnel_show.jpg tunnel_show%}


2. ngrok 
``` bash
ngrok http <port>
```

- **Claude Desktop加入Connector**
接著在Claude Desktop把剛才暴露的URL加入Connector。
{% asset_image add_custom_connector.jpg add_custom_connector%}

因為是測試的MCP，先選No sign-in
{% asset_image no_sign_in.jpg no_sign_in%}

加入成功!
{% asset_image add_success.jpg add_success%}

呼叫測試MCP Tool
```csharp
    [McpServerTool]
    [Description("Generates a random number between the specified minimum and maximum values.")]
    public int GetRandomNumber(
        [Description("Minimum value (inclusive)")] int min = 7533967,
        [Description("Maximum value (exclusive)")] int max = 7533987)
    {
        return Random.Shared.Next(min, max);
    }
```
{% asset_image call_mcp_tool.jpg call_mcp_tool%}
#
#
#

## 3.後續注意事項
測試成功後，也別忘記下面的事情：
1. **要記得補上身分驗證**
2. **要記得把對外暴露的url刪掉**
你可以指定Tunnel ID做刪除
``` bash
devtunnel delete <通道ID>
```

也可以全部刪除
``` bash
devtunnel delete-all
```
