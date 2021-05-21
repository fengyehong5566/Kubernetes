# service

service:  
　　定义了外界访问一组特定Pod的方式。  
	Service有自己的IP和端口，Service为Pod提供了负载均衡。  
	Service一个应用服务抽象，定义了Pod逻辑集合和访问这个Pod集合的策略。  
	Service代理Pod集合对外表现是为一个访问入口，分配一个集群IP地址，来自这个IP的请求将负载均衡转发后端Pod中的容器。  
	Service通过Lable Selector选择一组Pod提供服务。  
	注：  
　　k8s运行容器(Pod)和访问容器(Pod)这两项任务分别由Controller和Service执行


example:
```
   apiVersion	<string>
   kind	<string>
   metadata	<Object>
      annotations	<map[string]string>
      clusterName	<string>
      creationTimestamp	<string>
      deletionGracePeriodSeconds	<integer>
      deletionTimestamp	<string>
      finalizers	<[]string>
      generateName	<string>
      generation	<integer>
      labels	<map[string]string>
      managedFields	<[]Object>
         apiVersion	<string>
         fieldsType	<string>
         fieldsV1	<map[string]>
         manager	<string>
         operation	<string>
         time	<string>
      name	<string>
      namespace	<string>
      ownerReferences	<[]Object>
         apiVersion	<string>
         blockOwnerDeletion	<boolean>
         controller	<boolean>
         kind	<string>
         name	<string>
         uid	<string>
      resourceVersion	<string>
      selfLink	<string>
      uid	<string>
   spec	<Object>
      clusterIP	<string>
      externalIPs	<[]string>
      externalName	<string>
      externalTrafficPolicy	<string>
      healthCheckNodePort	<integer>
      ipFamily	<string>
      loadBalancerIP	<string>
      loadBalancerSourceRanges	<[]string>
      ports	<[]Object>
         name	<string>
         nodePort	<integer>
         port	<integer>
         protocol	<string>
         targetPort	<string>
      publishNotReadyAddresses	<boolean>
      selector	<map[string]string>
      sessionAffinity	<string>
      sessionAffinityConfig	<Object>
         clientIP	<Object>
            timeoutSeconds	<integer>
      type	<string>
   status	<Object>
      loadBalancer	<Object>
         ingress	<[]Object>
            hostname	<string>
            ip	<string>
```


