# Windows系统的Golang多版本管理方案

之前在我的 Windows 电脑安装 golang 环境，都是直接去官网下载 .msi 文件，然后双击安装。

但是每次升级golang版本时，都需要先把之前的版本卸载，然后下载新版本的 .msi 文件。 比较麻烦，而且多版本不能共存。

今天发现了一个新工具：[mise](https://github.com/jdx/mise)。
官网上说可以解决 golang 的多版本管理，而且不光 golang，python、Java等等都支持，今天来试试。

## 安装 mise {id="mise_1"}

<procedure title="安装步骤" id="install-mise">
    <p>我选择的方式是手工安装。</p>
    <step>
        <p>去官方的 <a href="https://github.com/jdx/mise/releases">Github</a> release 页，找到最新版的Windows安装包。</p>
        <img src="mise_github_release.png" alt="选择zip安装包" border-effect="line"/>
    </step>
    <step>
        <p>下载 .zip 安装包后解压，我解压到了 D 盘。</p>
    </step>
    <step>
        <p>将 mise.exe 可执行文件所在目录，添加至系统环境变量的 PATH 中，我添加的路径是 <code>D:\mise\bin</code> 。</p>
    </step>
</procedure>

接下来 mise 命令就可以用了。

## 修改 mise 默认下载目录 {id="mise_2"}

在下载 golang 之前，我们先修改一下 mise 的下载路径，因为它默认下载到 C盘。

<procedure title="修改步骤" id="modify-mise-env">
    <step>
        <p>创建一个文件夹 <code>D:\mise\data</code> ，我要让 mise 把 golang 下载到这个目录下。</p>
    </step>
    <step>
        <p>然后修改 mise 的环境变量，打开 PowerShell，在命令行输入以下命令:</p>
        <code-block lang="powershell">
            # 先用这个命令创建 powershell 的配置文件
            if (-not (Test-Path $profile)) { New-Item $profile -Force }
            # 打开 powershell 的配置文件
            Invoke-Item $profile
        </code-block>
        <tip>
            请注意至少使用 PowerShell7 版本以上，如果本机安装的是 Windows PowerShell 5，请在微软应用商店安装最新版 PowerShell7
        </tip>
    </step>
    <step>
        <p>系统会自动打开记事本，在里面输入以下命令，然后保存退出：</p>
        <code-block lang="powershell"><![CDATA[
            $env:MISE_DATA_DIR = "D:\mise\data"
            (&mise activate pwsh) | Out-String | Invoke-Expression
        ]]>
        </code-block>
        <img src="mise_power_shell_env.png" alt="修改mise环境变量" border-effect="line"/>
        <tip>
            环境变量 <code>MISE_DATA_DIR</code> 即代表 golang 下载目录
        </tip>
    </step>
</procedure>

## 下载 golang {id="mise_3"}

<procedure title="下载步骤" id="install-golang">
    <step>
        <p>首先查看一下可下载的 golang 版本有哪些：</p>
        <img src="mise_v.png" alt="查看可用 golang 版本" border-effect="line"/>
    </step>
    <step>
        <p>下载 golang 1.26.3 版本，执行命令 <code>mise use -g go@1.26.3</code></p>
        <img src="mise_golang_download.png" alt="下载 golang" border-effect="line"/>
    </step>
    <step>
        <p>可以看到 golang 1.26.3 已经可以使用了，但是默认的 <code>GOPATH</code> 指向的还是 C 盘。最后一步修改 <code>GOPATH</code> 指向路径，将 <code>E:\GOPATH</code> 添加到用户环境变量</p>
        <img src="mise_gopath.png" alt="下载 golang" border-effect="line"/>
    </step>
</procedure>

大功告成，之前下载的 .msi 版本 golang 可以直接卸载啦。