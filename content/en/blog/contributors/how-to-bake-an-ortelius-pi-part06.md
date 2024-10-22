---
date: 2024-09-14
title: "How to Bake an Ortelius Pi Part 6 | Cloud Dev At Home With Localstack"
linkTitle: "How to Bake an Ortelius Pi Part 6 | Cloud Dev At Home With Localstack"
author: Sacha Wharton
---

<!-- <div class="col-center">
<img src="/images/orteliuspi-part3.png" alt="raspberry-pi-4b" height="300px" width="650px" />
</div>
<p></p> -->

- [Introduction](#introduction)
- [Gimlet GitOps Infrastructure](#gimlet-gitops-infrastructure)
  - [Localstack](#localstack)
  - [Deploy Localstack](#deploy-localstack)
  - [Helm-Repository | Localstack](#helm-repository--localstack)
  - [Helm-Release | Localstack](#helm-release--localstack)
  - [Helm Chart Configuration Highlights](#helm-chart-configuration-highlights)
  - [Fluxcd is doing the following under the hood | Localstack](#fluxcd-is-doing-the-following-under-the-hood--localstack)
  - [Kubernetes check | Localstack](#kubernetes-check--localstack)
  - [Traefik](#traefik)
- [AWS and Localstack CLI Configuration](#aws-and-localstack-cli-configuration)
- [Localstack Tools](#localstack-tools)
- [Conclusion](#conclusion)
- [Next Steps](#next-steps)

### Introduction

In [part 5](https://ortelius.io/blog/2024/08/10/how-to-bake-an-ortelius-pi-part-5-ortelius-marries-jenkins/) we deployed [Jenkins](https://www.jenkins.io/) on our Kubernetes cluster and configured integration with [Ortelius](https://ortelius.io/) and [GitHub](https://github.com/) and built a demo application to demonstrate Ortelius's ability to record it.

In part 6 we will deploy [LocalStack](https://www.localstack.cloud/) and expose the endpoints through [Traefik](https://traefik.io/). We will use the AWS cli and the Localstack wrapper `awslocal` to create and list S3 buckets. To make using profiles with the cli easier we will use [Granted](https://www.granted.dev/) created by Commonfate. This will give us our very own cloud dev environment at home without all the cash burning worries and security headaches.

### Gimlet GitOps Infrastructure

#### Localstack

In today's cloud-centric world, developing and testing applications that rely on cloud services often presents unique challenges. Developers typically need access to various cloud environments like AWS to test their code. However, setting up and managing these cloud environments can be cumbersome, costly, and time-consuming, especially for frequent testing or when multiple cloud services are involved.

Enter LocalStack, a powerful tool that provides a fully functional local cloud environment. LocalStack emulates the core AWS services, such as S3, Lambda, DynamoDB, and many others, enabling developers to run and test their cloud applications directly on their local machines without needing an active AWS account or network access.

#### Deploy Localstack

Right lets get stuck in and deploy Localtack using Gimlet, Fluxcd, Helm and a sprig of GitOps.

- Kubectl quick reference guide [here](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
- Helm cheat sheet [here](https://helm.sh/docs/intro/cheatsheet/)
- Localstack on GitHub [here](https://github.com/localstack/)
- Localstack docs [here](https://www.jenkins.io/doc)
- Localstack integrations [here](https://docs.localstack.cloud/integrations/)
- Localstack academy [here](https://docs.localstack.cloud/academy/)
- Localstack tutorials [here](https://docs.localstack.cloud/tutorials/)
- Localstack applications [here](https://docs.localstack.cloud/applications/)
- Localstack extensions [here](https://docs.localstack.cloud/user-guide/extensions/)
- Localstack Helm Chart on ArtifactHub [here](https://artifacthub.io/packages/helm/localstack/localstack)

#### Helm-Repository | Localstack

- Lets add the Localstack Helm repository
- A Helm repository is a collection of Helm charts that are made available for download and installation
- Helm repositories serve as centralised locations where Helm charts can be stored, shared, and managed
- Create a file called `localstack.yaml` in the helm-repositories directory and paste the following YAML

```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: localstack
  namespace: infrastructure
spec:
  interval: 60m
  url: https://localstack.github.io/helm-charts
```

#### Helm-Release | Localstack

- Lets create a Helm release for LocalStack
- A Helm release is an instance of a Helm chart running in a Kubernetes cluster
- Each release is a deployment of a particular version of a chart with a specific configuration
- Create a file called `localstack.yaml` in the helm-releases directory and paste the following YAML

#### Helm Chart Configuration Highlights

```yaml
    image:
      repository: localstack/localstack # Replace this with localstack/localstack-pro if you have a subscription
      tag: "latest"
      pullPolicy: IfNotPresent
```

```yaml
    replicaCount: 3
```

```yaml
    mountDind:
      enabled: true
      forceTLS: true
      image: "docker:20.10-dind"
```

```yaml
    extraEnvVars:
      - name: GATEWAY_LISTEN
        value: "0.0.0.0:4566"
      - name: LOCALSTACK_API_KEY
        value: "<your api key"
      - name: LOCALSTACK_AUTH_TOKEN
        value: "<your auth token>"
```

```yaml
    ingress:
      enabled: true
      annotations: {}
      ## @param ingress.ingressClassName Set the name of the class to use
      ##
      ingressClassName: "traefik" # Only replace this if you are not using Traefik
      hosts:
        - host: localstack.pangarabbit.com # Replace with your domain
```

```yaml
    persistence:
      ## @param persistence.enabled Enable persistence using Persistent Volume Claims
      ##
      enabled: true # Set to false to disable persistent volumes
      ## @param persistence.storageClass Persistent Volume storage class
      ## If defined, storageClassName: <storageClass>
      ## If set to "-", storageClassName: "", which disables dynamic provisioning
      ## If undefined (the default) or set to null, no storageClassName spec is set, choosing the default provisioner
      ##
      storageClass: "nfs-csi-localstack" # Replace with your storage class
      ## @param persistence.accessModes [array] Persistent Volume access modes
      ##
      accessModes:
        - ReadWriteOnce
      ## @param persistence.size Persistent Volume size
      ##
      size: 8Gi
```

```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: localstack
  namespace: infrastructure
spec:
  interval: 60m
  releaseName: localstack
  chart:
    spec:
      chart: localstack
      version: 0.6.16 # Simply change the version to upgrade
      sourceRef:
        kind: HelmRepository
        name: localstack
      interval: 10m
  values:
    # Default values for LocalStack
    # Declare variables to be passed into your templates

    replicaCount: 3

    ## @param updateStrategy.type deployment strategy type
    ## ref: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy
    ## NOTE: Set it to `Recreate` if you use a PV that cannot be mounted on multiple pods
    ##
    updateStrategy:
      type: RollingUpdate

    image:
      repository: localstack/localstack-pro
      tag: "latest"
      pullPolicy: IfNotPresent

    imagePullSecrets: []
    nameOverride: ""
    fullnameOverride: ""

    ## @param extraDeploy Extra objects to deploy (value evaluated as a template)
    ##
    extraDeploy: []

    ## Add additional annotations to every resource
    extraAnnotations: {}

    serviceAccount:
      # Specifies whether a service account should be created
      create: true
      # Annotations to add to the service account
      annotations: {}
      # The name of the service account to use.
      # If not set and create is true, a name is generated using the fullname template
      name: ""

    role:
      # Specifies whether a role & rolebinding with pods / * permissions should be created for the service account
      # Necessary for kubernetes lambda executor
      create: true
      # Annotations to add to the role and rolebinding
      annotations: {}
      # The name of the role and rolebinding to use.
      # If not set and create is true, a name is generated using the fullname template
      name: ""

    podLabels: {}

    podAnnotations: {}

    podSecurityContext:
      {}
      # fsGroup: 2000

    securityContext:
      {}
      # capabilities:
      #   drop:
      #   - ALL
      # readOnlyRootFilesystem: true
      # runAsNonRoot: true
      # runAsUser: 1000

    debug: false

    ## @param command Allows you to set an arbitrary command on startup (instead of the default entrypoint script)
    ##
    command: []

    ## @param dnsPolicy Allows you to set the Pod dnsPolicy.
    ## The default is actually ClusterFirst. Uncomment this to avoid a circular DNS path that will
    ## cause the LocalStack instance to crash.
    ## Ref: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy
    # dnsPolicy: "Default"

    startServices: ""
    # Comma-separated list of AWS CLI service names which are the only ones allowed to be used (other services will then by default be prevented from being loaded).

    # kinesisErrorProbability: 0.0

    # lambdaExecutor: ""

    # This will enable the Docker daemon binding and allow
    # Localstack to provide Lambdas and other AWS services
    # who got container runtime dependencies
    mountDind:
      enabled: true
      forceTLS: true
      image: "docker:20.10-dind"

    ## All the parameters from the configuation can be added using extraEnvVars.
    ## Ref. https://docs.localstack.cloud/references/configuration/
    extraEnvVars:
      - name: GATEWAY_LISTEN
        value: "0.0.0.0:4566"
      - name: LOCALSTACK_API_KEY
        value: "<your api key"
      - name: LOCALSTACK_AUTH_TOKEN
        value: "your auth token"
    ##   - name: SERVICES
    ##     value: "serverless,sqs,es"
    livenessProbe:
      initialDelaySeconds: 0
      periodSeconds: 10
      timeoutSeconds: 1
      successThreshold: 1
      failureThreshold: 3
    readinessProbe:
      initialDelaySeconds: 0
      periodSeconds: 10
      timeoutSeconds: 1
      successThreshold: 1
      failureThreshold: 3

    service:
      type: ClusterIP
      annotations:
      ipFamilies: []
      ipFamilyPolicy: ""
      externalTrafficPolicy: ""
      edgeService:
        name: edge
        targetPort: 4566
        nodePort: 31566
      externalServicePorts:
        start: 4510
        end: 4560
        ## @param service.externalServicePorts.nodePortStart specifies the starting node ports the serviceports are mapped to
        ##   has to be in the node port range configured. See https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport
        # nodePortStart: 31510
      ## @param service.dnsService Enables or disables the exposure of the LocalStack DNS
      ##
      dnsService: false
      ## @param service.clusterIP sets a static clusterIP. This is useful alongside the LocalStack DNS setup
      ##   see https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/#configuration-of-stub-domain-and-upstream-nameserver-using-coredns for an example of DNS delegation in Coredns
      ##
      clusterIP: ""

    ingress:
      enabled: true
      annotations: {}
      ## @param ingress.ingressClassName Set the name of the class to use
      ##
      ingressClassName: "traefik"
      hosts:
        - host: localstack.pangarabbit.com
          paths:
            - path: /
              pathType: ImplementationSpecific
      # tls:
      #   - secretName: wildcard-pangarabbit-com-tls
      #     hosts:
      #       - localstack.pangarabbit.com

    persistence:
      ## @param persistence.enabled Enable persistence using Persistent Volume Claims
      ##
      enabled: true
      ## @param persistence.storageClass Persistent Volume storage class
      ## If defined, storageClassName: <storageClass>
      ## If set to "-", storageClassName: "", which disables dynamic provisioning
      ## If undefined (the default) or set to null, no storageClassName spec is set, choosing the default provisioner
      ##
      storageClass: "nfs-csi-localstack" # Add your storage class here
      ## @param persistence.accessModes [array] Persistent Volume access modes
      ##
      accessModes:
        - ReadWriteOnce
      ## @param persistence.size Persistent Volume size
      ##
      size: 8Gi
      ## @param persistence.dataSource Custom PVC data source
      ##
      dataSource: {}
      ## @param persistence.existingClaim The name of an existing PVC to use for persistence
      ##
      existingClaim: ""
      ## @param persistence.subPath The name of a volume's sub path to mount for persistence
      ##
      subPath: ""
      ## @param persistence.annotations Annotations to be added to PVC
      ##
      annotations: {}

    resources:
      {}
      # We usually recommend not to specify default resources and to leave this as a conscious
      # choice for the user. This also increases chances charts run on environments with little
      # resources, such as Minikube. If you do want to specify resources, uncomment the following
      # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
      # limits:
      #   cpu: 100m
      #   memory: 128Mi
      # requests:
      #   cpu: 100m
      #   memory: 128Mi

    # All settings inside the lambda values section are only applicable to the new v2 lambda provider
    lambda:
      # The lambda runtime executor.
      # Depending on the value, LocalStack will execute lambdas either in docker containers or in kubernetes pods
      # The value "kubernetes" depends on the service account and pod role being activated
      executor: "docker"
      # Image prefix for the kubernetes lambda images. The images will have to be pushed to that repository.
      # Only applicable when executor is set to "kubernetes"
      # Example: python3.9 runtime -> localstack/lambda-python:3.9
      image_prefix: "localstack/lambda-"
      # Timeout for spawning new lambda execution environments.
      # After that timeout, the environment (in essence pod/docker container) will be killed and restarted
      # Increase if spawning pods / docker containers takes longer in your environment
      environment_timeout: 60
      # Labels which will be assigned to the kubernetes pods spawned by the kubernetes executor.
      # They will be set on all spawned pods.
      # Only applicable when executor is set to "kubernetes"
      labels: {}
      # labels:
      #   label1: value1
      #   label2: value2
      #   label3: value3
      #
      # Security context to be set on the kubernetes pods spawned by the kubernetes executor.
      # It will be set on all spawned pods.
      # Only applicable when executor is set to "kubernetes"
      security_context: {}
      # security_context:
      #   runAsUser: 1000
      #   fsGroup: 1000
      #   label3: value3

    nodeSelector: {}

    tolerations: []

    affinity: {}

    # Mount /etc/localstack/init/ready.d to run startup scripts with
    # {{ template "localstack.fullname" . }}-init-scripts-config configMap
    enableStartupScripts: false

    # Add startup scripts content used by {{ template "localstack.fullname" . }}-init-scripts-config configMap
    # to run at localstack startup
    # startupScriptContent: |
    # awslocal s3 mb s3://testbucket
    # awslocal sqs create-queue --queue-name test-queue
    startupScriptContent: ""

    # @param volumes Extra volumes to mount
    volumes: []
    #  - hostPath:
    #      path: <HOST_PATH>
    #    name: <VOLUME_NAME>

    # @param volumeMounts Extra volumes to mount
    volumeMounts: []
    #  - name: <VOLUME_NAME>
    #    mountPath: <CONTAINER_PATH>
    #    readOnly: true

    ## @param priorityClassName Allows you to set the priorityClassName for the pod
    ## The default is not to set any priorityClassName
    # priorityClassName: ""
```

- Lets git it

```shell
git add .
git commit -m "localstack deploy"
git push
```

#### Fluxcd is doing the following under the hood | Localstack

- Helm repo add

```shell
helm repo add localstack-charts https://localstack.github.io/helm-charts --force-update
```

- Helm install localstack

```shell
helm install localstack localstack-charts/localstack
```

#### Kubernetes check | Localstack

- Kubectl switch to the infrastructure namespace

```shell
kubectl config set-context --current --namespace=infrastructure
```

- Kubectl show me the pods for Localstack

```shell
kubectl get pods -n infrastructure | grep localstack
```

<div class="col-left">
<img src="/images/how-to-bake-an-ortelius-pi/part06/01-localstack-pods.png" alt="localstack pods"/>
</div>
<p></p>

- Kubectl show me the persistent claims

```shell
kubectl get pvc | grep localstack
```

<div class="col-left">
<img src="/images/how-to-bake-an-ortelius-pi/part06/12-localstack-pvc.png" alt="localstack pvc"/>
</div>
<p></p>

- Now that we have deployed Localstack we will expose the Localstack endpoints with Traefik

#### Traefik

- Open your Traefik Helm Chart from the `helm-releases` directory for your infrastructure repo that Gimlet created
- For example mine is `gitops-pangarabbit-dev-infra/helm-releases`
- Add the following entrypoint after the `websecure` entrypoint in your `traefik.yaml` `helm-releases` file

```yaml
      localstack:
        ## -- Enable this entrypoint as a default entrypoint. When a service doesn't explicitly set an entrypoint it will only use this entrypoint.
        # asDefault: true
        port: 4566
        # hostPort: 4566
        containerPort: 4566
        expose:
          default: true
        exposedPort: 4566
        ## -- Different target traefik port on the cluster, useful for IP type LB
        # targetPort: 80
        ## -- The port protocol (TCP/UDP)
        protocol: TCP
```

- Lets git it

```shell
git add .
git commit -m "traefik localstack entrypoint"
git push
```

- Navigate to the `traefik-dynamic-config.yaml` file in the `manifests/` directory and add the `IngressRoute` for Localstack and push your changes

```yaml
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: localstack
  namespace: infrastructure
spec:
  entryPoints:
    - localstack
  routes:
    - match: Host("localstack.pangarabbit.com") # Replace this with your domain
      kind: Rule
      services:
        - name: localstack
          namespace: infrastructure
          kind: Service
          port: 4566
```

- Lets git it

```shell
git add .
git commit -m "traefik localstack ingressroute"
git push
```

- You should see the Localstack endpoint represented on your Traefik dashboard

<div class="col-left">
<img src="/images/how-to-bake-an-ortelius-pi/part06/11-traefik-localstack-endpoint.png" alt="traefik localstack endpoint"/>
</div>
<p></p>

- Don't forget to add a DNS record for the Localstack domain name you used
- If everything went well you should be able to curl the Localstack endpoint with the domain name that you chose for example mine is `https://localstack.pangarabbit.com`
- Open our terminal and use curl to test the Localstack endpoint
- You don't need the `:4566` port at the end of the url as Traefik takes care of that when we created the `IngressRoute`

```shell
 curl -vvv https://localstack.pangarabbit.com
```

- The output below shows you the endpoint is alive and well and secure

```shell
* Host localstack.pangarabbit.com:443 was resolved.
* IPv6: (none)
* IPv4: 192.168.0.151
*   Trying 192.168.0.151:443...
* Connected to localstack.pangarabbit.com (192.168.0.151) port 443
* ALPN: curl offers h2,http/1.1
* (304) (OUT), TLS handshake, Client hello (1):
*  CAfile: /etc/ssl/cert.pem
*  CApath: none
* (304) (IN), TLS handshake, Server hello (2):
* (304) (IN), TLS handshake, Unknown (8):
* (304) (IN), TLS handshake, Certificate (11):
* (304) (IN), TLS handshake, CERT verify (15):
* (304) (IN), TLS handshake, Finished (20):
* (304) (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / AEAD-CHACHA20-POLY1305-SHA256 / [blank] / UNDEF
* ALPN: server accepted h2
* Server certificate:
*  subject: CN=pangarabbit.com
*  start date: Aug  1 16:47:29 2024 GMT
*  expire date: Oct 30 16:47:28 2024 GMT
*  subjectAltName: host "localstack.pangarabbit.com" matched cert's "*.pangarabbit.com"
*  issuer: C=US; O=Let's Encrypt; CN=R10
*  SSL certificate verify ok.
* using HTTP/2
* [HTTP/2] [1] OPENED stream for https://localstack.pangarabbit.com/
* [HTTP/2] [1] [:method: GET]
* [HTTP/2] [1] [:scheme: https]
* [HTTP/2] [1] [:authority: localstack.pangarabbit.com]
* [HTTP/2] [1] [:path: /]
* [HTTP/2] [1] [user-agent: curl/8.7.1]
* [HTTP/2] [1] [accept: */*]
> GET / HTTP/2
> Host: localstack.pangarabbit.com
> User-Agent: curl/8.7.1
> Accept: */*
>
* Request completely sent off
< HTTP/2 200
< alt-svc: h3=":8443"; ma=2592000
< content-type: text/plain; charset=utf-8
< date: Sat, 14 Sep 2024 20:00:22 GMT
< server: TwistedWeb/24.3.0
< content-length: 0
<
* Connection #0 to host localstack.pangarabbit.com left intact
```

### AWS and Localstack CLI Configuration

- Lets test to see if we can create a S3 bucket using the Localstack AWS wrapper command line tool called `awslocal` and the AWS cli
- To make our lives less painful switching profiles install Granted created by Commonfate
- Use the Granted `getting started` doc [here](https://docs.commonfate.io/granted/getting-started)
- Please install the Localstack AWS cli wrapper tool [here](https://docs.localstack.cloud/user-guide/integrations/aws-cli/)
- For more information on installation and configuring the AWS cli go [here](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- Create fictitious credentials in `.aws/credentials` for Localstack

```shell
[base]
aws_access_key_id=fake
aws_secret_access_key=fake
```

- Create a profile in `.aws/config`

```shell
[profile localstack]
endpoint_url       = https://localstack.pangarabbit.com:4566 # Replace with your endpoint domain name
services           = localstack
region             = us-east-1
output             = yaml
cli_history        = enabled
cli_pager          = bat
credential_process = granted credential-process --profile=localstack

[services localstack]
s3               =
endpoint_url     = https://localstack.pangarabbit.com:4566 # Replace with your endpoint domain name
addressing_style = path
```

- Install [Granted]((https://docs.commonfate.io/granted/getting-started)) for your OS flavour
- Test that Granted was installed correctly

```shell
granted -v
```

- Assume your first role [here](https://docs.commonfate.io/granted/usage/assuming-roles)
- You role name will be `localstack`
- After completing the first time setup you should be able to just type `assume` on the command line and a selection will pop up

```shell
assume
```

<div class="col-left">
<img src="/images/how-to-bake-an-ortelius-pi/part06/02-assume.png" alt="assume"/>
</div>
<p></p>

- S3 bucket creation

```shell
aws s3api create-bucket --bucket ortelius-bucket-aws --endpoint https://localstack.pangarabbit.com
```

```shell
awslocal s3api create-bucket --bucket ortelius-bucket-awslocal --endpoint https://localstack.pangarabbit.com
```

<div class="col-left">
<img src="/images/how-to-bake-an-ortelius-pi/part06/03-s3-create-bucket.png" alt="s3 create bucket"/>
</div>
<p></p>

- Lets list s3 buckets with `awslocal`

```shell
awslocal s3api list-buckets --endpoint https://localstack.pangarabbit.com --profile localstack-pangarabbit
```
<div class="col-left">
<img src="/images/how-to-bake-an-ortelius-pi/part06/05-s3-list-awslocal.png" alt="s3 list awslocal"/>
</div>
<p></p>

- Lets list s3 buckets with `aws`

```shell
aws s3api list-buckets --endpoint https://localstack.pangarabbit.com --profile localstack-pangarabbit
```

<div class="col-left">
<img src="/images/how-to-bake-an-ortelius-pi/part06/04-s3-list-aws.png" alt="s3 list aws"/>
</div>
<p></p>

### Localstack Tools

You can find LocalStack tools [here](https://docs.localstack.cloud/user-guide/tools/). One of these tools, **Localstack Desktop**, provides a GUI that allows you to view and manage your Localstack resources and services on your local machine. It also lets you add custom endpoints that you've created.

To get Localstack Desktop, [sign up for Localstack](https://app.localstack.cloud/sign-up) with your browser, where you'll have access to the Localstack portal. From there, you can manage everything related to your Localstack setup, including viewing instances and adding custom endpoints.

You can access the LocalStack portal at this URL: `https://app.localstack.cloud/dashboard`. Once inside, look for the **LocalStack Desktop** menu item.

<div class="col-left">
<img src="/images/how-to-bake-an-ortelius-pi/part06/06-localstack-desktop.png" alt="localstack desktop"/>
</div>
<p></p>

- Localstack custom endpoint in the browser

<div class="col-left">
<img src="/images/how-to-bake-an-ortelius-pi/part06/07-localstack-custom-endpoint-browser.png" alt="localstack custom endpoint browser"/>
</div>
<p></p>

- Localstack custom endpoint in Localstack Desktop

<div class="col-left">
<img src="/images/how-to-bake-an-ortelius-pi/part06/08-localstack-custom-endpoint-desktop.png" alt="localstack custom endpoint desktop"/>
</div>
<p></p>

- Localstack services in the browser

<div class="col-left">
<img src="/images/how-to-bake-an-ortelius-pi/part06/09-localstack-services-browser.png" alt="localstack services browser"/>
</div>
<p></p>

- Localstack services in Localstack Desktop

<div class="col-left">
<img src="/images/how-to-bake-an-ortelius-pi/part06/10-localstack-services-desktop.png" alt="localstack services desktop"/>
</div>
<p></p>

### Conclusion

***FYI make sure you backup your persistent volumes on the NFS server***.

You now have a local working cloud to develop against, test applications and learn safely. If I missed any steps or something needs correction please ping me so I can make any fixes. This illustrates how you can deploy LocalStack and publish the endpoint through Traefik.

Happy alien hunting.....

### Next Steps

[How to Bake an Ortelius Pi | Part 7 | Observability with Netdata](https://ortelius.io/blog/2024/08/10/how-to-bake-an-ortelius-pi-part-7-observability-with-Netdata/)

{{< blocks/section color=white >}}

<h2 class="text-left">Meet the Author</h2>
<hr>

{{< blocks/feature_dual >}}

Learn More About:
- [Sacha Wharton](https://linktr.ee/sachawharton)

{{< /blocks/feature_dual >}}
{{< blocks/feature_dual >}}

<div style="position:relative;left:-60%">
<img src="/images/sacha.jpg" alt="Sachawharton" height="400px" width="400px" />
</div>

{{< /blocks/feature_dual >}}
{{< /blocks/section >}}
