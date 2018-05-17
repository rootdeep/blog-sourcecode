---

title: "helm 自带私有仓库使用方法"
draft: false
date: "2018-05-16T14:03:57+08:00"
categories: "kubernetes"
tags: ["kubernetes", "helm"]

---

安装helm 客户端后，使用 helm serve 命令可以在本地启动一个私有仓库。
通过helm repo list 可以看到一个仓库名，及仓库地址为:

```local   http://127.0.0.1/charts```

helm serve 的启动命令参数如下：
```
Usage:
  helm serve [flags]

Flags:
      --address string     address to listen on (default "127.0.0.1:8879")
      --repo-path string   local directory path from which to serve charts
      --url string         external URL of chart repository

Global Flags:
      --debug                           enable verbose output
      --home string                     location of your Helm config. Overrides $HELM_HOME (default "/root/.helm")
      --host string                     address of Tiller. Overrides $HELM_HOST
      --kube-context string             name of the kubeconfig context to use
      --tiller-connection-timeout int   the duration (in seconds) Helm will wait to establish a connection to tiller (default 300)
      --tiller-namespace string         namespace of Tiller (default "kube-system")
```

一般的，如果不添加任何命令参数启动本地仓库，helm 会以http://localhost:8879 为服务地址，以~/.helm/repository/local 为本地仓库来保存chart模板。在使用"helm serve" 启动本地仓库时，会在~/.helm/repository/local 下生成一个index.yaml 文件。该文件记录了chart 模板的元数据，类似于yum 源repodata 目录下数据（或者apt 源的Package.gz 文件）。

如果local 目录下没有chart 包，生成的index.yaml 的内容就是空的。所以，在启动本地服务仓库之前，可以把制作好的本地chart 压缩包（*.tgz）方法该目录下，这样生成的index.yaml 文件就有是内容的。 

使用localhost:8879 为服务地址时，只能在本地浏览器访问仓库。通过添加 helm serve 的启动参数，可以使其在机器的物理IP上工作，并且，也可以设置其他的仓库位置。例如使用/var/repo 作为仓库来保存模板可以按照以下步骤实现：
```
mkdir /var/repo
# 复制chart包到 /var/repo 目录
cp chart1.tzg  /var/repo
helm serve --address IP:8879 --repo-path  /var/repo # 会生成index.yaml 文件
```
最后一步，添加本地仓库到helm 的仓库目录中

```helm repo add {repo_name}   http://IP:8879```

例如：
```
[root@lzg-02 ~]# helm repo add my_local  http://192.168.51.252:8879
"my_local" has been added to your repositories
```

通过```helm repo list``` 可以看到新加的仓库 及仓库地址。
例如：
```
[root@lzg-02 ~]# helm repo list
NAME    	URL                                             
stable  	https://kubernetes-charts.storage.googleapis.com
my_local	http://192.168.51.252:8879 
```
此外，如果先启动了本地仓库，再往仓库加镜像时，先把chart包复制到仓库目录下，再
通过```helm repo index```命令重新生成index.yalm 文件。具体示例如下：

```helm repo index /var/repo --url http://192.168.51.252:8879```

通过浏览器访问 http://IP:8879 可以看到新添加的chart 包。


删除一个仓库： helm repo remove {repo_name}
