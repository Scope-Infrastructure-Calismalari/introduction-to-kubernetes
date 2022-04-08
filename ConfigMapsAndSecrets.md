# Introduction to Kubernetes

Editors: **Kaan Keskin, Sezen Erdem**

Date: November 2021

Available at: https://github.com/kaan-keskin/introduction-to-kubernetes

**Resources:**

> - Kubernetes Documentation - https://kubernetes.io/docs/home/
> - Kubernetes in Action - Marko Lukša - Manning Publications
> - Kubernetes Fundamentals (LFS258) - Timothy Serewicz - The Linux Foundation
> - Kubernetes for Developers (LFD259) - Timothy Serewicz - The Linux Foundation
> - Kubernetes Lecture Notes - California Institute of Technology
> - Certified Kubernetes Application Developer (CKAD) Study Guide - Benjamin Muschko - O'Reilly Media
> - Getting Started with Kubernetes - Sander van Vugt - Addison-Wesley Professional

**LEGAL NOTICE: This document is created for educational purposes, and it can not be used for any commercial intentions. If you find this document useful in any means please support the original authors for ethical reasons.** 

[Return to the README page.](README.md)

# ConfigMaps and Secrets

One of the fundamental principles of continuous delivery is to build a binary artifact just once for a single SCM commit. A binary artifact should then be stored in a binary repository—for example, in the commercial product JFrog Artifactory. An automated process would then download the artifact if needed for deployment to a target environment. Typical target runtime environments include staging or production. The method of building an artifact just once prevents accidental mistakes if you were to rebuild it per environment and increases the overall confidence level when shipping the software to the customer. Configuration data needed for each of those environments can be injected into the runtime environment. Environment variables can help with this task, but there are other options.

Kubernetes dedicates two primitives to defining configuration data: the ConfigMap and the Secret. **Both primitives are completely decoupled from the lifecycle of a Pod, which enables you to change their configuration data values without necessarily having to redeploy the Pod.** In essence, ConfigMaps and Secrets store a set of key-value pairs. Those key-value pairs can be injected into a container as environment variables, or they can be mounted as a Volume.

<img src=".\images\pod-consuming-configuration-data.png"/>

Figure above shows an example Pod that decided to consume data from a ConfigMap as a volume mount and a Secret as environment variables.

What’s the difference between a ConfigMap and a Secret? They’re almost identical in purpose and structure, although Secrets are better suited for storing sensitive data like passwords, API keys, or SSL certificates because they store their values encoded in Base64. Let me also mention the security aspect of Secrets. Base64 only encodes a value, but it doesn’t encrypt it. Therefore, anyone with access to its value can swiftly decode it. **A Secret is distributed only to the nodes running Pods that actually require access to it. Moreover, Secrets are stored in memory and are never written to a physical storage.**

## ConfigMaps

A ConfigMap is an API object used to store non-confidential data in key-value pairs. It is a map containing key/value pairs with the values ranging from short literals to full config files. A ConfigMap allows you to decouple environment-specific configuration from your container images, so that your applications are easily portable.
The whole point of an app’s configuration is to keep the config options that vary between environments, or change frequently, separate from the application’s source code. 

An application doesn’t need to read the ConfigMap directly or even know that it exists. Pods can consume ConfigMaps as environment variables, command-line arguments, or as configuration files in a volume.

<img src=".\images\p3_configmaps.jpg"/>

### Portable Data with ConfigMaps

A similar API resource to Secrets is the ConfigMap, except the data is not encoded. In keeping with the concept of decoupling in Kubernetes, using a ConfigMap decouples a container image from configuration artifacts. 

They store data as sets of key-value pairs or plain configuration files in any format. The data can come from a collection of files or all files in a directory. It can also be populated from a literal value. 

A ConfigMap can be used in several different ways. A container can use the data as environmental variables from one or more sources. The values contained inside can be passed to commands inside the pod. A Volume or a file in a Volume can be created, including different names and particular access modes. In addition, cluster components like controllers can use the data. ​

Let's say you have a file on your local filesystem called **config.js**. You can create a ConfigMap that contains this file. The **configmap** object will have a data section containing the content of the file:

```shell
$ kubectl get configmap foobar -o yaml 

kind: ConfigMap 
apiVersion: v1 
metadata: 
    name: foobar 
data: 
    config.js: | 
         { 
...
```

ConfigMaps can be consumed in various ways:
- Pod environmental variables from single or multiple ConfigMaps
- Use ConfigMap values in Pod commands
- Populate Volume from ConfigMap
- Add ConfigMap data to specific path in Volume
- Set file names and access mode in Volume from ConfigMap data
- Can be used by system components and controllers.

### Creating ConfigMaps

You can create a ConfigMap imperatively with a single command: **kubectl create configmap**. As part of the command, you have to provide a mandatory command-line flag that points to the source of the data. 

Kubernetes distinguishes four different options:
- Literal values, which are key-value pairs as plain text.
- A file that contains key-value pairs and expects them to be environment variables.
- A file with arbitrary contents.
- A directory with one or many files.

**Literal values**: By passing literals to the kubectl command:

```shell
kubectl create configmap myconfigmap --from-literal=foo=bar --from-literal=bar=baz --from-literal=one=two
```

**Single file with environment variables**:

```shell
$ kubectl create configmap db-config --from-env-file=config.env

configmap/db-config created
```

**Single file**:

```shell
$ kubectl create configmap db-config --from-file=config.txt

configmap/db-config created
```

**Directory containing files**:

```shell
$ kubectl create configmap db-config --from-file=app-config

configmap/db-config created
```

Alternatively, you can also create the ConfigMap declaratively. 

**By defining in a YAML file**:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # property-like keys; each key maps to a simple value
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"
  # file-like keys
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5    
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true    
```

<img src=".\images\p3_configmaps_createfromdirectory.jpg"/>

### Using ConfigMaps

You can use ConfigMaps as environment variables or using a volume mount. They must exist prior to being used by a Pod, unless marked as optional. They also reside in a specific namespace.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
  namespace: default
data:
  database_url: jdbc:postgresql://localhost/test
  user: fred
```

There are four different ways that you can use a ConfigMap to configure a container inside a Pod:

1. Inside a container command and args
2. Environment variables for a container
3. Add a file in read-only volume, for the application to read
4. Write code to run inside the Pod that uses the Kubernetes API to read a ConfigMap

These different methods lend themselves to different ways of modeling the data being consumed. For the first three methods, the kubelet uses the data from the ConfigMap when it launches container(s) for a Pod. The fourth method means you have to write code to read the ConfigMap and its data.

### Consuming a ConfigMap as Environment Variables

In the case of environment variables, your pod manifest will use the valueFrom key and the configMapKeyRef value to read the values. 

**Once the ConfigMap has been created, it can be consumed by one or many Pods in the same namespace.**

Here, we’re exploring how to inject the key-value pairs of a ConfigMap as environment variables. The Pod definition shown in example below references the ConfigMap named backend-config and injects the key-value pairs as environment variables with the help of envFrom.configMapRef.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configured-pod
  namespace: default
spec:
  containers:
  - image: nginx:1.19.0
    name: app
    envFrom:
    - configMapRef:
        name: backend-config
```

It’s important to mention that the attribute envFrom does not automatically format the key to conform to typical conventions used by environment variables (all caps letters, words separated by the underscore character). The attribute simply uses the keys as-is. After creating the Pod, you can inspect the injected environment variables by executing the remote Unix command env inside of the container:

```shell
$ kubectl exec configured-pod -- env

...
database_url=jdbc:postgresql://localhost/test
user=fred
...
```

Sometimes, key-value pairs do not conform to typical naming conventions for environment variables or can’t be changed without impacting running services. **You can redefine the keys used to inject an environment variable into a Pod with the valueFrom attribute. Example below turns the key database_url into DATABASE_URL and the key user into USERNAME.**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configured-pod
spec:
  containers:
  - image: nginx:1.19.0
    name: app
    env:
    - name: DATABASE_URL
      valueFrom:
        configMapKeyRef:
          name: backend-config
          key: database_url
    - name: USERNAME
      valueFrom:
        configMapKeyRef:
          name: backend-config
          key: user
```

The resulting environment variables available to the container now follow the typical conventions for environment variables:

```shell
$ kubectl exec configured-pod -- env

...
DATABASE_URL=jdbc:postgresql://localhost/test
USERNAME=fred
...
```

### Mounting a ConfigMap as Volume

Most programming languages can resolve and use environment variables to control the runtime behavior of an application. Especially when dealing with a long list of configuration data, it might be preferable to access the key-value pairs from the filesystem of the container.

**A ConfigMap can be mounted as Volume. The application would then read those key-value pairs from the filesystem with an expected mount path. Kubernetes represents every key in the ConfigMap as a file. The value becomes the content of the file.**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configured-pod
spec:
  containers:
  - image: nginx:1.19.0
    name: app
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: backend-config
```

The volumes attribute specifies the Volume to use. As you can see in the code snippet, it points to the name of the ConfigMap. The name of the Volume is relevant for adding the mount path using the volumeMounts attribute. Here, we’re pointing to the mount path /etc/config.

To verify the expected behavior, open an interactive shell. As shown in the following terminal output, the directory contains the files database_url and user. Those filenames correspond to the keys of the ConfigMap. The file contents represent their corresponding value in the ConfigMap:

```shell
$ kubectl exec -it configured-pod -- /bin/sh

# ls -1 /etc/config
database_url
user

# cat /etc/config/database_url
jdbc:postgresql://localhost/test

# cat /etc/config/user
fred
```

### Example

Here's an example Pod that uses values from game-demo to configure a Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: alpine
      command: ["sleep", "3600"]
      env:
        # Define the environment variable
        - name: PLAYER_INITIAL_LIVES # Notice that the case is different here
                                     # from the key name in the ConfigMap.
          valueFrom:
            configMapKeyRef:
              name: game-demo           # The ConfigMap this value comes from.
              key: player_initial_lives # The key to fetch.
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: ui_properties_file_name
      volumeMounts:
      - name: config
        mountPath: "/config"
        readOnly: true
  volumes:
    # You set volumes at the Pod level, then mount them into containers inside that Pod
    - name: config
      configMap:
        # Provide the name of the ConfigMap you want to mount.
        name: game-demo
        # An array of keys from the ConfigMap to create as files
        items:
        - key: "game.properties"
          path: "game.properties"
        - key: "user-interface.properties"
          path: "user-interface.properties"
```

## Secrets

A Secret is an object that contains a small amount of sensitive data such as a password, a token, or a key. Using a Secret means that you don't need to include confidential data in your application code. Secrets are similar to ConfigMaps but are specifically intended to hold confidential data.

A Secret can be used with a Pod in three ways:

1. As files in a volume mounted on one or more of its containers.
2. As container environment variable.
3. By the kubelet when pulling images for the Pod.

Pods can access local data using volumes, but there is some data you don't want readable to the naked eye. Passwords may be an example. Someone reading through a YAML file may read a password and remember it. Using the Secret API resource, the same password could be encoded or encrypted. A casual reading would not give away the password.

You can create, get, or delete secrets:

```shell
$ kubectl get secrets 
```

Secrets can be encoded manually or via '**kubectl create secret**':​

```shell
$ kubectl create secret generic --help

$ kubectl create secret generic mysql --from-literal=password=root
```

A secret is not encrypted, only base64-encoded, by default. You must create an **EncryptionConfiguration** with a key and proper identity. Then, the kube-apiserver needs the **--encryption-provider-config** flag set to a previously configured provider, such as **aescbc** or **ksm**. Once this is enabled, you need to recreate every secret, as they are encrypted upon write. 

Multiple keys are possible. Each key for a provider is tried during decryption. The first key of the first provider is used for encryption. To rotate keys, first create a new key, restart (all) kube-apiserver processes, then recreate every secret. 

You can see the encoded string inside the secret with **kubectl**. The secret will be decoded and be presented as a string saved to a file. The file can be used as an environmental variable or in a new directory, similar to the presentation of a volume.

A secret can be made manually as well, then inserted into a YAML file:

```shell
$ echo LFTr@1n | base64

TEZUckAxbgo= 

$ vim secret.yaml

apiVersion: v1
kind: Secret
metadata: 
  name: lf-secret
data: 
  password: TEZUckAxbgo=
```

Prior to Kubernetes v1.18 secrets (and configMaps) were automatically updated. This could lead to issues. If a configuration was updated and a Pod restarted,​​​​​​​ it may be configured differently than other replicas. In newer versions these objects can be made immutable.

### Creating Secret

You can create a Secret imperatively with a single command: **kubectl create secret**. Similar to the command for creating a ConfigMap, you will have to provide an additional subcommand and a configuration option. It’s mandatory to spell out the subcommand right after the Kubernetes resource type secret. 

You can select from one of the options:
- **generic**: Creates a secret from a file, directory, or literal value.
- **docker-registry**: Creates a secret for use with a Docker registry.
- **tls**: Creates a TLS secret.

In most cases, you will likely deal with the type generic, which provides the same command-line options to point to the source of the configuration data as kubectl create configmap:

- Literal values, which are key-value pairs as plain text.
- A file that contains key-value pairs and expects them to be environment variables.
- A file with arbitrary contents.
- A directory with one or many files.

Let’s have a look at some command-line usage examples for creating a Secret with the type generic. All values you feed into the command will be stored internally Base64 encoded. For example, the value s3cre! turns into czNjcmUh.

#### Literal values

```shell
$ kubectl create secret generic db-creds --from-literal=pwd=s3cre!

secret/db-creds created
```

#### File containing environment variables

```shell
$ kubectl create secret generic db-creds --from-env-file=secret.env

secret/db-creds created
```

#### SSH key file

```shell
$ kubectl create secret generic ssh-key --from-file=id_rsa=~/.ssh/id_rsa

secret/db-creds created
```

#### Declarative

Of course, you can always take the declarative route, but there’s a little catch. You have to Base64-encode the configuration data value yourself when using the type Opaque. 

How can you do so? One way to encode and decode a value is the Unix command-line tool **base64**. Alternatively, you can use websites like Base64 Encode. The following example uses the command-line tool:

```shell
$ echo -n 's3cre!' | base64

czNjcmUh
```

You can now plug in the value under the data section with a corresponding key, as shown in example below.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
type: Opaque
data:
  pwd: czNjcmUh
```

Another example shown below:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  USER_NAME: YWRtaW4=
  PASSWORD: MWYyZDFlMmU2N2Rm
```

```shell
kubectl apply -f mysecret.yaml
```

Refer to the Kubernetes documentation for other types assignable to a Secret that do not require explicit Base64-encoding. One example is the type kubernetes.io/basic-auth, which represents credentials needed for basic authentication.

### Using Secrets

Secrets can be mounted as data volumes or exposed as environment variables to be used by a container in a Pod.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
```

### Using Secrets via Environment Variables

Consuming the key-value pairs of a Secret as environment variables from a container works almost exactly the same way as it does for a ConfigMap. There’s only one difference: instead of using envFrom.configMapRef, you’d use envFrom.secretRef.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configured-pod
spec:
  containers:
  - image: nginx:1.19.0
    name: app
    envFrom:
    - secretRef:
        name: db-creds
```

**It’s important to understand that the container will make the environment variable available in a Base64-decoded value.** In turn, your application running in the container will not have to implement Base64-decoding logic:

```shell
$ kubectl exec configured-pod -- env

...
pwd=s3cre!
...
```

A secret can be used as an environmental variable in a Pod. You can see one being configured in the following example: 

```yaml
... 
spec: 
  containers: 
  - image: mysql:5.5
    name: dbpod
    env: 
    - name: MYSQL_ROOT_PASSWORD 
      valueFrom: 
        secretKeyRef: 
          name: mysql 
          key: password 
```

**There is no limit to the number of Secrets used, but there is a 1MB limit to their size. Each secret occupies memory, along with other API objects, so very large numbers of secrets could deplete memory on a host.**

They are stored in the **tmpfs** storage on the host node, and are only sent to the host running Pod. All volumes requested by a Pod must be mounted before the containers within the Pod are started. So, a secret must exist prior to being requested.

### Mounting Secrets as Volumes

In practice, you will see Secrets mounted as Volumes fairly often, especially in the context of making an SSH private key available to the container. 

Example below assumes you’ve created a Secret named ssh-key with the key id_rsa. First, create a Volume by pointing it to the name of the Secret with secret.secretName. Note that the attribute referencing the name is different than for a ConfigMap; for Secrets, it’s called secretName. Next, mount the Volume by its name and provide a mount path.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configured-pod
spec:
  containers:
  - image: nginx:1.19.0
    name: app
    volumeMounts:
    - name: secret-volume
      mountPath: /var/app
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: ssh-key
```

Secrets mounted as Volume will expose its values in Base64-decoded form. You can easily verify the value by opening an interactive shell and printing the contents of the file '/var/app/id_rsa' to standard output:

```shell
$ kubectl exec -it configured-pod -- /bin/sh

# ls -1 /var/app
id_rsa

# cat /var/app/id_rsa
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,8734C9153079F2E8497C8075289EBBF1
...
-----END RSA PRIVATE KEY-----
```

You can also mount secrets as files using a volume definition in a pod manifest. The mount path will contain a file whose name will be the key of the secret created with the **kubectl create secret** step earlier.

```yaml
... 
spec: 
    containers: 
    - image: busybox 
      command: 
        - sleep 
        - "3600" 
      volumeMounts: 
      - mountPath: /mysqlpassword 
        name: mysql 
      name: busy 
    volumes: 
    - name: mysql 
        secret: 
            secretName: mysql
```

Once the pod is running, you can verify that the secret is indeed accessible in the container:

```shell
$ kubectl exec -ti busybox -- cat /mysqlpassword/password

LFTr@1n
```
