# 原文:[Harbor + Kubernetes = 自托管容器注册表](https://loft.sh/blog/harbor-kubernetes-self-hosted-container-registry/)

自我托管绝不是一个新概念。多年来，工程师和IT管理员一直在研究他们如何自我托管工具。主要是因为自我托管是唯一可行的解决方案，因为云提供商尚未开始受到关注。但是，即使在上述云提供商的普及最近增加，许多人仍然考虑是否应该自助。

[Harbor](https://goharbor.io/) 如果您想为Docker映像自我托管集装箱注册表，则是解决方案。它最初是在VMware内部开发的，但此后一直是 [adopted by CNCF](https://blogs.vmware.com/opensource/2018/07/31/cncf-adopts-project-harbor/)。如今，它是一种开源工具，旨在为用户提供尽可能多的功能，同时保持自由。在本教程中，您将向您展示如何将其启动并在内部运行[Kubernetes](https://kubernetes.io/).

## [#](https://loft.sh/blog/harbor-kubernetes-self-hosted-container-registry/#what-is-harbor)什么是港口？

本质上，港口是一个自托管的集装箱注册表。这意味着，您可以自己做，而不是依靠诸如GCP或Azure之类的提供商来托管您的图像。这可以使您的所有基础架构内部保持内部，并使您不再依赖其他来源。对于许多人来说，这可能是一个有价值的特征。有些组织更喜欢自己托管事物，而其他组织则喜欢医疗行业 [HIPAA](https://en.wikipedia.org/wiki/Health_Insurance_Portability_and_Accountability_Act) 限制 - *有*自己托管。

除了能够亲自托管Docker注册表之外，Harbor还具有各种其他不错的功能，其中许多功能与改善安全性有关。在港口托管的图像后，您可以设置[漏洞扫描](https://goharbor.io/docs/2.0.0/administration/vulnerability-scanning/) 为了确保您了解图像中存在的所有漏洞。这是通过开源项目[Trivy]完成的(https://github.com/aquasecurity/trivy) 和[Clair](https://github.com/coreos/clair)。您可以使用严重性级别来决定允许使用哪些图像，例如限制任何包含严重漏洞的图像。最重要的是，港口还为通用供应链安全，签名图像等提供了支持。

## [#](https://loft.sh/blog/harbor-kubernetes-self-hosted-container-registry/#why-use-harbor)为什么要使用港口？

虽然上一部分中提到的功能听起来确实像它们为任何部署增加了很多东西，但您可能仍然想知道为什么要使用Harbor而不是其他工具。好吧，有很多原因强调其价值与其他工具相比。第一个可能的原因是您需要对注册表进行更多的控制 - 您希望能够将其完全配置为自己的喜好。尽管许多提供商确实提供了很多配置，但您通常会遇到提供商提供的部署方式。有了一个自托管解决方案，您就是决定如何部署事物的人。

您可能还会发现港口拥有一些您在其他任何地方都找不到的功能。例如，拥有Dev，QA和Prod的单独注册表并不少见。这也可以通过港口完成，但港口也使他们可以轻松地进行凝聚力管理。它甚至提供了在存储库之间同步图像的能力，从而为您提供了通过不同部署阶段促进图像的直接方法。现在，请继续阅读以了解如何使其运行。

## [#](https://loft.sh/blog/harbor-kubernetes-self-hosted-container-registry/#setting-up-harbor)建立港口

在开始建立港口之前，请确保拥有所有先决条件：

* [Kubectl](https://kubernetes.io/docs/reference/kubectl/kubectl/) 用于管理群集
* [Helm](https://helm.sh/) 3用于安装港口
* [Minikube](https://minikube.sigs.k8s.io/docs/)作为本地Kubernetes群集的创建工具
* [VirtualBox](https://minikube.sigs.k8s.io/docs/drivers/virtualbox/) 作为Minikube的驱动程序

### [#](https://loft.sh/blog/harbor-kubernetes-self-hosted-container-registry/#installing-harbor)Installing Harbor

在典型的生产情况下，您将在GCP或AWS之类的东西上运行Kubernetes群集，但对于教程来说，这相当重。在这里，您将使用Minikube；一种旨在在本地旋转kubernetes簇的工具。安装了Minikube，通过运行以下命令来启动新集群：

```bash
$ minikube start --vm-driver virtualbox
```

这将需要一段时间，但是完成命令后，您将拥有一个运行的kubernetes群集。在这一点上，您需要启用Ingress附加组件，以便您可以访问港口安装，因此请运行以下操作：

```bash
$ minikube addons enable ingress
```

到现在为止，您应该将所有内容配置在Minikube中。接下来，您将努力为Harbor安装Helm图表，但首先需要将存储库添加到Helm：

```bash
$ helm repo add harbor https://helm.goharbor.io
```

添加了存储库后，您可以通过运行以下内容安装Helm图表：

```bash
$ helm install my-release harbor/harbor
```

在这一点上，您需要等待所有豆荚进入运行状态。通过运行检查 `kubectl get pods`。您可能会看到其中一些正在失败，这将发生，因为它们彼此依赖。您可能需要至少等待十五到二十分钟才能准备好。运行后，运行`minikube ip` 获取Minikube群集的IP地址。

您现在需要使用此IP来更改您的 `/etc/hosts`文件。港口的默认URL是 `https://core.harbor.domain`，但是您需要确保将其输入到浏览器的地址栏中时，它会解决到群集。通过将以下两条新线输入到您的身上来做到这一点`/etc/hosts` 文件：

```bash
<ip-of-minikube>	core.harbor.domain
<ip-of-minikube>	notary.harbor.domain
```

现在你应该能够去 [https://core.harbor.domain](https://core.harbor.domain/) 并使用默认用户名和密码登录：

```
username: admin
password: Harbor12345
```

![Harbor login screen](https://loft.sh/blog/images/content/kasper-siig-harbor-login.png?nf_resize=fit&w=1040)

### [#](https://loft.sh/blog/harbor-kubernetes-self-hosted-container-registry/#configuring-the-docker-client)Configuring the Docker Client

在这一点上，您有一个可以开始使用的主动港口安装。但是，这并不意味着您准备将其用作注册表。您仍然需要安装港口证书，以确保您的PC与注册表之间的通信不会被阻止。

首先，您需要配置Docker守护程序以在Minikube中使用。假设您正在运行Linux或OSX，则可以运行以下操作：

```bash
$ eval $(minikube docker-env)
```

这将配置您的环境变量以指向Minikube Docker守护程序。接下来，您需要从Kubernetes Secrets获得证书：

```bash
kubectl -n harbor get secrets harbor-ingress -o jsonpath="{.data['ca\.crt']}" | base64 -D > harbor-ca.crt
```

> 请注意，这使用 `base64 -D`，而在Linux上，它将使用 `base64 -d`.

此时，您有一个`harbor-ca.crt` 包含证书的文件。要将其安装在Docker守护程序中，您需要首先将证书复制到Minikube VM中：

```bash
$ scp -o IdentitiesOnly=yes -i $(minikube ssh-key) harbor-ca.crt docker@$(minikube ip):./harbor-ca.crt
```

复制证书后，您可以通过登录Minikube VM并安装它来安装证书：

```bash
$ minikube ssh
$ sudo mkdir -p /etc/docker/certs.d/core.harbor.domain
$ sudo cp harbor-ca.crt /etc/docker/certs.d/core.harbor.domain
```

最后，运行 `exit` 回到您的普通终端。在这一点上，已安装证书，您现在可以测试通过登录和推动图像来按预期工作的一切工作：

```bash
# Log into the registry
$ docker login core.harbor.domain --username=admin --password Harbor12345

# Pull an image from Docker Hub
$ docker pull nginx

# Tag the image, so it's ready to be pushed
$ docker tag nginx core.harbor.domain/library/nginx:latest

# Push the image to the registry
$ docker push core.harbor.domain/library/nginx:latest
```

## [#](https://loft.sh/blog/harbor-kubernetes-self-hosted-container-registry/#conclusion)Conclusion

如果您成功地达到了这一点，现在您可以为Harbor提供一个主动安装，并且可以开始使用它作为您自己的私人注册表。这使您可以完全控制您的注册表及其部署方式。最重要的是，您可以在此开源工具中获得所有可用功能，例如漏洞扫描和图像复制。您可以自己测试该工具，并确切地了解如何使用它，而不是阅读所有功能的列表。

无论您运行私人注册表的原因是什么，您现在都知道它是如何完成的。从这里开始，您可以更恰当地比较自托管的两种情况与托管解决方案。弄清楚自我托管解决方案的优势可以给您带来什么优势，以及它们是否超过了托管的解决方案。

*Photo by [Cameron Venti](https://unsplash.com/@ventiviews?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/containers?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)*

