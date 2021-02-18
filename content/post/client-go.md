---
title: "K8S源码剖析之Client-Go"
date: 2020-08-08T13:40:57+08:00
lastmod: 2020-08-08T13:40:57+08:00
draft: false
keywords: ["kubernetes"]
description: "client-go源码理解"
tags: ["k8s源码","client-go"]
menu: "kubernetes"
categories: ["kubernetes"]
author: "Jeremy-boo"


contentCopyright: '<a rel="license noopener" href="https://en.wikipedia.org/wiki/Wikipedia:Text_of_Creative_Commons_Attribution-ShareAlike_3.0_Unported_License" target="_blank">Creative Commons Attribution-ShareAlike License</a>'
---

<!--more-->
# 1. k8s之client-go源码剖析
> Face with a smile, not to complain.
##  Client-Go源码结构
源码结构:
![client-go](/images/client-go.png)
## 1.1 Client-Go之五大客户端对象
> client-go 是一个调用Kubernetes集群资源对象API的客户端，通过Client-go实现对Kubernetes集群中资源对象(Deploment,Service,Pod,Secret等)的CRUD操作。官方文档：https://github.com/kubernetes/client-go 
- 1.1.1 基础常识
client-go 操作k8s集群其实是通过kubeconfig配置信息或者k8s-api-server的地址连接指定的kubernetes-api-server，连接k8s例子：

1.获取依赖包

    go get github.com/kubernetes/client-go

2.连接k8s(确保本地有kubeconfig配置文件)

    package main

    import (
        "flag"
        "fmt"
        "os"
        "path/filepath"
        "time"

        metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
        "k8s.io/client-go/kubernetes"
        "k8s.io/client-go/tools/clientcmd"
    )
    func main() {
      var kubeconfig *string
      if home := homeDir(); home != "" {
            kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional) absolute path to the kubeconfig file")
        } else {
            kubeconfig = flag.String("kubeconfig", "", "absolute path to the kubeconfig file")
        }
      flag.Parse()
      // uses the current context in kubeconfig
      config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
      if err != nil {
          panic(err.Error())
      }
      // creates the clientset
      clientset, err := kubernetes.NewForConfig(config)
      if err != nil {
          panic(err.Error())
      }
      // get pods list
      for {
          pods, err := clientset.CoreV1().Pods("").List(metav1.ListOptions{})
          if err != nil {
              panic(err.Error())
          }
          fmt.Printf("There are %d pods in the cluster\n", len(pods.Items))
          time.Sleep(10 * time.Second)
      }
    }

    func homeDir() string {
      if h := os.Getenv("HOME"); h != "" {
        return h
      }
      return os.Getenv("USERPROFILE") // windows
    }

- 1.1.2 RESTClient
RESTClient是最基础的客户端，它是基于对Http Request进行了封装，实现RESTful风格的API，其他Client都是基于RESTClient实现的,实际上就是一个http client
通过参数(master的url或者kubeconfig路径)和BuildConfigFronFlags

       // config      
        config,err := clientcmd.BuildConfigFormFlags("masterURL",*kubeconfig)
        // restClient
        restClient,err := rest.RESTClientFor(config)

源码路径：k8s.io/client-go/rest/client.go

    // RESTClient imposes common Kubernetes API conventions on a set of resource paths.
    // The baseURL is expected to point to an HTTP or HTTPS path that is the parent
    // of one or more resources.  The server should return a decodable API resource
    // object, or an api.Status object which contains information about the reason for
    // any failure.
    //
    // Most consumers should use client.New() to get a Kubernetes API client.
    type RESTClient struct {
        // base is the root URL for all invocations of the client
        base *url.URL
        // versionedAPIPath is a path segment connecting the base URL to the resource root
        versionedAPIPath string

        // contentConfig is the information used to communicate with the server.
        contentConfig ContentConfig

        // serializers contain all serializers for underlying content type.
        serializers Serializers

        // creates BackoffManager that is passed to requests.
        createBackoffMgr func() BackoffManager

        // TODO extract this into a wrapper interface via the RESTClient interface in kubectl.
        Throttle flowcontrol.RateLimiter

        // Set specific behavior of the client.  If not set http.DefaultClient will be used.
        Client *http.Client
    }

源码路径：k8s.io/client-go/rest/client.go
RESTClient.Interface中方法如下：

      // Interface captures the set of operations for generically interacting with Kubernetes REST apis.
      type Interface interface {
          GetRateLimiter() flowcontrol.RateLimiter
          Verb(verb string) *Request
          Post() *Request
          Put() *Request
          Patch(pt types.PatchType) *Request
          Get() *Request
          Delete() *Request
          APIVersion() schema.GroupVersion
      }


- Discovery Client
discovery client(发现客户端)，用于发现kube-apiserver支持的资源组、资源版本以及资源信息等(即Versions,Groups,Resources)

      // config      
      config,err := clientcmd.BuildConfigFormFlags("masterURL",*kubeconfig)
      // discovery client
      discoveryClient, err := discovery.NewDiscoveryClientForConfig(config)
      
- 1.1.3 Dynamic Client
dynamic client能够处理kubernetes集群中的所有资源对象，包括用户自定义资源

      // config      
      config,err := clientcmd.BuildConfigFormFlags("masterURL",*kubeconfig)
      // dynamic client
      dynamicClient, err := dynamic.NewForConfig(config)
- 1.1.4 ClientSet Client
ClientSet是在RESTClient基础上封装的对k8s资源(Resoures)和版本(Versions)操作管理的客户端，只能够处理k8s内置资源(Pod，Namespace等)

      
      // config      
      config,err := clientcmd.BuildConfigFormFlags("masterURL",*kubeconfig)
      // clientSet client
      clientSet,err := kubernetes.NewForCofig(cofnig)
      // get pods
      pods, err := clientset.CoreV1().Pods("").List(metav1.ListOptions{})


clientset的结构体

      // Clientset contains the clients for groups. Each group has exactly one
      // version included in a Clientset.
      type Clientset struct {
          *discovery.DiscoveryClient
          admissionregistrationV1alpha1 *admissionregistrationv1alpha1.AdmissionregistrationV1alpha1Client
          appsV1beta1                   *appsv1beta1.AppsV1beta1Client
          appsV1beta2                   *appsv1beta2.AppsV1beta2Client
          authenticationV1              *authenticationv1.AuthenticationV1Client
          authenticationV1beta1         *authenticationv1beta1.AuthenticationV1beta1Client
          authorizationV1               *authorizationv1.AuthorizationV1Client
          authorizationV1beta1          *authorizationv1beta1.AuthorizationV1beta1Client
          autoscalingV1                 *autoscalingv1.AutoscalingV1Client
          autoscalingV2beta1            *autoscalingv2beta1.AutoscalingV2beta1Client
          batchV1                       *batchv1.BatchV1Client
          batchV1beta1                  *batchv1beta1.BatchV1beta1Client
          batchV2alpha1                 *batchv2alpha1.BatchV2alpha1Client
          certificatesV1beta1           *certificatesv1beta1.CertificatesV1beta1Client
          coreV1                        *corev1.CoreV1Client
          extensionsV1beta1             *extensionsv1beta1.ExtensionsV1beta1Client
          networkingV1                  *networkingv1.NetworkingV1Client
          policyV1beta1                 *policyv1beta1.PolicyV1beta1Client
          rbacV1                        *rbacv1.RbacV1Client
          rbacV1beta1                   *rbacv1beta1.RbacV1beta1Client
          rbacV1alpha1                  *rbacv1alpha1.RbacV1alpha1Client
          schedulingV1alpha1            *schedulingv1alpha1.SchedulingV1alpha1Client
          settingsV1alpha1              *settingsv1alpha1.SettingsV1alpha1Client
          storageV1beta1                *storagev1beta1.StorageV1beta1Client
          storageV1                     *storagev1.StorageV1Client
      }


## 1.2 Client-Go之Informer机制
> Informer机制是保证kubernetes各个组件通过http协议通信消息的一致性、实时性、顺序性
#### 1.2.1 Informer机制核心组件以及运行原理图
1. Informer核心组件：
![informer](/images/informer.PNG)

2. Informer运行原理图：
> 图片来源：kubernetes源码剖析

![informer-runtime](/images/informer_runtime.PNG)

#### 1.2.2 

## 1.3 Clinet-Go之WorkQueue
## 1.4 Client-Go之事件管理器
## 1.5 k8s-dashboard对client-go的一些封装
## 1.6 Client-Go的基础操作
## 1.7 成长与思考
#### 1.7.1 client-go总结
client-go主要功能就是对k8s集群的各种资源进行操作，针对每种资源都封装成了相应的client提供调用，使用client-go时首先需要获取kubernetes配置信息，即kubeconfig，大致目录：$HOME/.kube/config。

> 调用过程如下：

加载kubeconfig配置文件-> resr.config -> clientset -> 具体的client(CoreV1Client)->具体的资源对象(pod)->RESTClient->http.Client→HTTP请求的发送及响应

通过clientset中不同的client和client中不同资源对象的方法实现对kubernetes中资源对象的增删改查等操作，常用的client有CoreV1Client、AppsV1beta1Client、ExtensionsV1beta1Client等。


