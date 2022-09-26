# 原文: [https://ruzickap.github.io/k8s-harbor/#links](https://ruzickap.github.io/k8s-harbor/#links)

[Harbor **(opens new window)**](https://goharbor.io/)is an open source cloud native registry that stores, signs, and scans container images for vulnerabilities.

![Harbor](https://ruzickap.github.io/k8s-harbor/assets/img/harbor-horizontal-color.ef644877.svg "Harbor")

港口通过提供信任，合规性，绩效和互操作性来解决共同的挑战。它填补了无法使用公共或基于云的注册表或希望在云中获得一致体验的组织和应用程序的空白。

* Demo GitHub存储库： [https://github.com/ruzickap/k8s-harbor**(opens new window)**](https://github.com/ruzickap/k8s-harbor)
* 演示网页: [https://ruzickap.github.io/k8s-harbor**(opens new window)**](https://ruzickap.github.io/k8s-harbor)
* 演示文稿git存储库: [https://github.com/ruzickap/k8s-harbor-presentation**(opens new window)**](https://github.com/ruzickap/k8s-harbor-presentation)
* 演示URL: [https://ruzickap.github.io/k8s-harbor-presentation**(opens new window)**](https://ruzickap.github.io/k8s-harbor-presentation)
* YouTube: [Harbor presentation in Czech language**(opens new window)**](https://youtu.be/niZJOM7ND24)
* Asciinema屏幕截图: [https://asciinema.org/a/253519**(opens new window)**](https://asciinema.org/a/253519)
* Asciinema屏幕截图 (45 minutes): [https://asciinema.org/a/278803**(opens new window)**](https://asciinema.org/a/278803)

## [#](https://ruzickap.github.io/k8s-harbor/#requirements)Requirements

* [ansible**(opens new window)**](https://ansible.com/)
* [awscli**(opens new window)**](https://aws.amazon.com/cli/)
* [AWS IAM Authenticator for Kubernetes**(opens new window)**](https://github.com/kubernetes-sigs/aws-iam-authenticator)
* [AWS account**(opens new window)**](https://aws.amazon.com/account/)
* [kubectl**(opens new window)**](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
* [eksctl**(opens new window)**](https://eksctl.io/)
* Kubernetes, Docker, Linux, AWS knowledge required

## [#](https://ruzickap.github.io/k8s-harbor/#objectives)Objectives

* 下载并安装港口到您的kubernetes群集

## [#](https://ruzickap.github.io/k8s-harbor/#lab-architecture)Lab Architecture

![Lab architecture](https://raw.githubusercontent.com/ruzickap/k8s-harbor-presentation/master/images/harbor_demo_architecture_diagram.svg?sanitize=true "Lab architecture")

## [#](https://ruzickap.github.io/k8s-harbor/#content)Content

* [Part 01 - Create EKS cluster](https://ruzickap.github.io/k8s-harbor/part-01/)
* [Part 02 - Install Helm](https://ruzickap.github.io/k8s-harbor/part-02/)
* [Part 03 - ingress-nginx + cert-manager installation](https://ruzickap.github.io/k8s-harbor/part-03/)
* [Part 04 - Harbor installation](https://ruzickap.github.io/k8s-harbor/part-04/)
* [Part 05 - Initial Harbor tasks](https://ruzickap.github.io/k8s-harbor/part-05/)
* [Part 06 - Harbor and Helm charts](https://ruzickap.github.io/k8s-harbor/part-06/)
* [Part 07 - Harbor and container images](https://ruzickap.github.io/k8s-harbor/part-07/)
* [Part 08 - Project settings](https://ruzickap.github.io/k8s-harbor/part-08/)
* [Part 09 - Clean-up](https://ruzickap.github.io/k8s-harbor/part-09/)

## [#](https://ruzickap.github.io/k8s-harbor/#links)Links

* Video:
  * [Intro to Harbor**(opens new window)**](https://youtu.be/Rs3zByxI8aY)
  * [Intro: Harbor - James Zabala &amp; Henry Zhang, VMware**(opens new window)**](https://youtu.be/RZQVBWwGa2s)
  * [Deep Dive: Harbor - Tan Jiang &amp; Jia Zou, VMware**(opens new window)**](https://youtu.be/OKj1XxtsTCo)
* Pages:
  * [Deploying Harbor Container Registry in Production**(opens new window)**](https://medium.com/@ikod/deploy-harbor-container-registry-in-production-89352fb1a114)
  * [How to install and use VMware Harbor private registry with Kubernetes**(opens new window)**](https://blog.inkubate.io/how-to-use-harbor-private-registry-with-kubernetes/)
  * [Use the Notary client for advanced users**(opens new window)**](https://docs.docker.com/notary/advanced_usage/)
  * [Signing Docker images with Notary server**(opens new window)**](https://werner-dijkerman.nl/2019/02/24/signing-docker-images-with-notary-server/)
  * [Handy API Harbor calls (in Chinese)**(opens new window)**](https://cloud.tencent.com/developer/article/1151425)
  * [Swagger Editor **(opens new window)**](https://editor.swagger.io/)+ Import [Harbor&#39;s swagger.yaml**(opens new window)**](https://raw.githubusercontent.com/goharbor/harbor/7b6e83090e26d171c0d0e0dacd14e2b61fab45e1/API/harbor/swagger.yaml)

![Harbor](https://raw.githubusercontent.com/cncf/artwork/ab42c9591f6e0fdccc62c7b88f353d3fdc825734/harbor/stacked/color/harbor-stacked-color.svg?sanitize=true "Harbor")

