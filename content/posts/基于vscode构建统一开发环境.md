---
title:  "基于 vscode 构建远程开发环境"
url:  "/vscode/dev"
date:  2023-06-06T09:52:06+08:00
description:  
keywords: 
  - 云原生
  - vscode
categories: 
  - 云原生
draft:  false
toc:  true
images: 
tags:  
  - 云原生
  - 工具
---

## 背景
我的主力操作系统是 Window，平常会在 `wsl` 下使用 Linux 的一些命令和脚本，相比之下更熟悉 Linux 命令，之前主要使用的 IDE 是 Jetbrains 全家桶，但最近由于同时打开三四个 IDE 导致内存吃不消（Jetbrains 属实是吃内存），且 `vscode` 编辑器的火热让我有了远程打造一套远程开发环境的想法，于是有了本文的记录和分享。

## vscode 技巧
如果是第一次使用 `vscode`, 建议学习一下 `vscode`  的使用技巧, 
[可以参考一下这篇官方文档](https://code.visualstudio.com/docs/getstarted/tips-and-tricks#_default-keyboard-shortcuts)。
从这里可以学到很多常用的快捷键，比如: 
- 最常用的命令面板: `Ctrl+Shift+P`
- 快速打开文件: `Ctrl+P`
- 向上方插入一行: `Ctrl+Shift+Enter`
- 向下方插入一行: `Ctrl+Enter`
- 快速选择相同内容: `Ctrl+D`
- 快速选择全部相同内容: `Ctrl+Shift+L`
- 快速选择全部相同单词: `Ctrl+F2`
- 多行操作，光标下移: `Ctrl+Alt+⬇`
- 多行操作，光标上移: `Ctrl+Alt+⬆`
- 多行操作，任务位置插入光标: `Alt+Click`
- 向上复制当前行: `Shift+Alt+⬆`
- 向下复制当前行: `Shift+Alt+⬇`
- 向上移动当前行: `Alt+⬆`
- 向下移动当前行: `Alt+⬇`
- 向上滚动页面: `Alt+⬆`
- 向下滚动页面: `Alt+⬇`
- 折叠代码: `Ctrl+Shift+[`
- 展开代码: `Ctrl+Shift+]`
- 添加行注释: `Ctrl+/`或者`Ctrl+K Ctrl+C`
- [更多参考](https://code.visualstudio.com/shortcuts/keyboard-shortcuts-windows.pdf):  `Ctrl+K Ctrl+R`

## 使用 WSL 作为开发环境
如果在 Window 系统上需要使用 Linux 环境或命令但有不想安装 Docker，那么 [WSL](https://code.visualstudio.com/docs/remote/wsl-tutorial) 就是首选了。
在 `vscode` 上使用 WSL 可以让你既能享受 `vscode` UI 的便利，又能在 Linux 上编译、调试代码, 而且使用上非常简单，只需在 `vscode` 左下角远程管理点击连接上 WSL 或者命令行的目标位置下执行 `code .` 即可打开一个 WSL 环境的 `vscode` 窗口，也可以通过万能的 `Ctr1+Shift+P` 快捷键打开命令面板后选择连接 WSL。

![](https://microsoft.github.io/vscode-remote-release/images/remote-wsl-open-code.gif)


## 使用远程服务器作为开发环境
本地开发机的资源和性能总是有限的，这时候我们需要借助高性能的服务器来开发测试，在 `vscode` 使用 [Remote-SSH](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh)
 连接远程服务器与连接本地 WSL 一样简单，使用 `Ctrl+Shift+P` 命令后选择 `Connect to Host`，按照提示填写服务器的用户、IP、SSH 监听端口以及密码之后就可以如同在本地一样，在远程服务器上编写、调试代码。从此再也不用担心运行耗性能的程序会让本地机器死机了，而且服务器的环境也可以更贴近真实的生产环境，及早发现一些环境导致的问题。也可以多人共享一个统一的开发环境，甚至维护同一份代码（需谨慎）。

![](https://microsoft.github.io/vscode-remote-release/images/ssh-readme.gif)

## 使用容器作为开发环境
相对于使用虚拟机/服务器作为开发环境，我更喜欢使用容器来作为开发环境，可以把整套开发环境一起打包，具备更高的可移植性，对于团队的新人更友好，省去很多安装开发环境的时间，在学习新技术的时候可以直接运行程序而不必担心影响开发环境。

在 `vscode` 中只需要安装 [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) 插件就可以快速启动一个容器来作为你的开发环境

![](https://microsoft.github.io/vscode-remote-release/images/remote-containers-readme.gif)

### 使用本地容器作为开发环境 
在使用本地容器开发环境之前，需要在本地先安装好 Dokcer,
然后使用命令面板执行 `Dev Containers: New Dev Container...` 来创建一个新的容器开发环境，默认情况下，`Dev Containers` 会将本地存在的文件夹作为一个存储卷挂载进容器后启动，自动安装完 `vscode server`, 最后本地 `vscode` UI 自动使用此开发环境重新加载窗口。

要做到上述流程，`Dev Containers` 是通过 `.devcontainer` 文件夹下的 `devcontainer.yaml` 来自定义配置的，如容器名称、镜像名称等,除此之外还可以有 `Dockerfile`、`Docker Componse` 等配置。

```yaml
{
	"name": "Go",
	// Or use a Dockerfile or Docker Compose file. More info: https://containers.dev/guide/dockerfile
	"image": "mcr.microsoft.com/devcontainers/go:0-1.19-bullseye",

	// Features to add to the dev container. More info: https://containers.dev/features.
	// "features": {},

	// Configure tool-specific properties.
	"customizations": {
		// Configure properties specific to VS Code.
		"vscode": {
			"settings": {},
			"extensions": [
				"streetsidesoftware.code-spell-checker"
			]
		}
	},

	// Use 'forwardPorts' to make a list of ports inside the container available locally.
	// "forwardPorts": [9000],

	// Use 'portsAttributes' to set default properties for specific forwarded ports. 
	// More info: https://containers.dev/implementors/json_reference/#port-attributes
	"portsAttributes": {
		"9000": {
			"label": "Hello Remote World",
			"onAutoForward": "notify"
		}
	}

	// Use 'postCreateCommand' to run commands after the container is created.
	// "postCreateCommand": "go version",

	// Uncomment to connect as root instead. More info: https://aka.ms/dev-containers-non-root.
	// "remoteUser": "root"
}
```
[完整的 `devcontainer.json` 示例](https://containers.dev/implementors/json_reference/)


### 在运行的普通容器中附加 `vscode-sever`
在命令面板中使用 `Dev Containers: Attach to Running Container...`, 然后选择正在运行的容器即可，[Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) 插件会自动在运行容器中启动 `vscode server`, 然后驱动本地 `vscode` 进行远程连接, 如果没有多余的设置，默认不会打开一个容器内的文件夹，需要手动选择打开文件夹，然后就可以进行正常的调试了，这种一般用于调试正在运行的服务代码，比较适合调试新技术等第三方代码的场景。

![](../..assets/vscode-server.png)


### 使用远程容器作为开发环境 
在本地使用容器作为开发环境比起直接使用本地环境或者 WSL 的方式更消耗资源，特别是使用多个开发环境的情况，而使用的容器作为开发环境本来就是为了隔离多个开发环境，使其彼此不相互影响，所以更好的方式是使用远程容器的方式来作为开发环境。

有两种方式可以达到这个目的：
1. 使用 [Remote-SSH](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh)
 插件连接（官方推荐，本地不需要安装 Docker）:
    - 需要提前在远程服务器上安装 Docker 服务
	 - 在 `vscode` 的配置文件中配置 `docker.environment` 变量并重新启动 `vscode`
	 ```yaml
    	docker.environment": {
    		HOST": "ssh://your-remote-user@your-remote-machine-fqdn-or-ip-here"
    }
    ```
2.  使用 Docker 客户端, 通过 `socket` 的方式进行连接(本地需要安装 Docker 和 `vscode` 插件)：
    - 需要提前在本地上安装 Docker 客户端 
    - 在 `vscode` 的配置文件中配置 `docker.environment` 变量并重新启动 `vscode`
    ```yaml
    "docker.environment": {
		"DOCKER_HOST": "tcp://your-remote-machine-fqdn-or-ip-here:port",
		"DOCKER_CERT_PATH": "/optional/path/to/folder/with/certificate/files",
		"DOCKER_TLS_VERIFY": "1" // or "0"
    }
	```
配置完成之后 [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) 插件在新建开发容器的时候就会默认使用配置的远程 Docker 服务
1.  使用 `Dev Containers:  New Dev Container` 命令在远程服务器打开一个新的开发容器
2.  或者使用 `Dev Containers: Open folder in Container` 命令打开项目路径，项目中需要有 `devcontainer` 文件夹相关的配置文件
	```yaml
	{
		"name": "vscode-remote-go",
		"image": "golang:1.20", // Or "dockerFile"
		"workspaceFolder": "/vscode-remote-go",
		"workspaceMount": "source=/mnt/e/chb/go/src/vscode-remote-go,target=/vscode-remote-go,type=bind,consistency=cached"
	}
	```
3. 也可以通过 `Dev Containers: Clone Repository in Container Volume` 或者 `Dev Containers: Clone Gighub Pull Request in Container Volume`  命令拉取远程仓库的方式来创建远程容器开发环境 
4. 以上几个命令执行后， `vscode` 会先自动借助 [Remote-SSH](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh)
 插件连接上远程服务器，然后创建存储卷并挂载、运行容器，最后使用本地 `vscode` UI 连接上容器中的 `vscode server` 并打开 `workdir`。	


## 使用远程K8S作为开发环境
除了上述的几种方式外，如果有 K8S 等基础设施，也可以通过 [Remote-Kubernetes](https://marketplace.visualstudio.com/items?itemName=okteto.remote-kubernetes) 插件来构建统一的开发环境。这个插件会使用 [Okteto 帮助我们通过简单的 `yaml` 配置，自动化地管理 K8S `pod` 。

在使用这个插件之前，我们需要先安装好 [Remote-SSH](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh)
 插件，配置好 [kubeconfig](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)。

编辑以下 `yaml` 文件之后，使用 `Okteto: Up` 命令即可在 K8S 集群中轻松启动一个开发环境，需要注意的是集群网络环境与镜像仓库要能联通, 在 `pod` 启动之前，`okteto` 会创建一个 `pvc` 并等待 `pvc` 绑定完成， 然后将当前文件夹挂载进入 `pod`。

[入门示例](https://www.okteto.com/blog/remote-kubernetes-development/)
```yaml
dev:
  vscode-remote-go:
    name: hello-world
    image: okteto/golang:1.16
    autocreate: true
    workdir: /okteto
    command: ["bash"]
    volumes:
      - /go/pkg/                # persist go dependencies
      - /root/.cache/go-build/  # persist go build cache
      - /root/.vscode-server    # persist vscode extensions
    securityContext:
      capabilities:
        add:
        - SYS_PTRACE            # required by the go debugger
    forward:
      - 8080:8080
    persistentVolume:
      enabled: true
    initContainer:
      image: okteto/bin:1.4.2
      resources:
        requests:
          cpu: 30m
          memory: 30Mi
        limits:
          cpu: 30m
          memory: 30Mi
# build:
#   vscode-remote-go:
#     image: okteto/golang:1.16
#     context: .

# deploy:
#   - kubectl apply -f k8s.yml
```
[完整示例](https://www.okteto.com/docs/reference/manifest/)

## References
- [Remote Explorer](https://marketplace.visualstudio.com/items?itemName=ms-vscode.remote-explorer)
- [WSL](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-wsl)
- [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)
- [Remote-SSH](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh)
- [Remote-Kubernetes](https://marketplace.visualstudio.com/items?itemName=okteto.remote-kubernetes)