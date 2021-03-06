# Secret

You don’t want sensitive information such as a database password or an API key kept around in clear text. Secrets provide you with a mechanism to use such information in a safe and reliable way with the following properties:

- Secrets are namespaced objects, that is, exist in the context of a namespace
- You can access them via a volume or an environment variable from a container running in a pod
- The secret data on nodes is stored in tmpfs volumes
- A per-secret size limit of 1MB exists
- The API server stores secrets as plaintext in etcd

![Kubernetes](https://img.shields.io/badge/Kubernetes-informational?logo=Kubernetes&color=blue&logoColor=white&style=for-the-badge&logoWidth=30)

![OpenShift](https://img.shields.io/badge/OpenShift-informational?logo=Red%20Hat%20Open%20Shift&color=black&logoColor=red&style=for-the-badge&logoWidth=30)


## Prerequisites

Having completed the following labs:

- [00 - Prerequisites](../00-Prerequisites/README.md)
- [02 - Provision the environment](../02-Provision_the_environment/README.md)
- [03 - OKD login](../03-OKD_login/README.md)
- [04 - Project](../04-Project/README.md)

Having logged in using the **developer** account:

```console
$ oc login -u developer -p developer https://api.crc.testing:6443     
Login successful.

You have one project on this server: "test"

Using project "test".
```

Make sure to use the **test** project.

```console
$ oc project test
Already on project "test" on server "https://api.crc.testing:6443".
```

## Secret as a file in the Container

Let’s create a file apikey that holds a (made-up) API key:

```console
$ echo -n "A19fh68B001j" > ./apikey.txt
```

This file can be used to create a Kubernetes/OpenShift Secret

```console
$ oc create secret generic apikey --from-file=./apikey.txt
secret/apikey created
```

Inspect the secret

```console
$ oc describe secrets/apikey
Name:         apikey
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
apikey.txt:  12 bytes
```

Now let’s use the secret in a pod via a volume:


```console
$ oc apply -f pod-secretvolume.yaml
pod/pod-secretvolume created
```

This pod declaration contains the following snippet, which takes a secret by name and injects it as a volume within the pod's container.

```yaml
...
  containers:
  - name: shell
    image: centos:7
    command:
      - "bin/bash"
      - "-c"
      - "sleep 10000"
    volumeMounts:
      - name: apikeyvol
        mountPath: "/tmp/apikey"
        readOnly: true
  volumes:
  - name: apikeyvol
    secret:
      secretName: apikey
...
```

If we now exec into the container we see the secret mounted at /tmp/apikey:

```console
$ oc exec -it pod-secretvolume -c shell -- bash
[root@pod-secretvolume /]# mount | grep apikey
tmpfs on /tmp/apikey type tmpfs (ro,relatime)
[root@pod-secretvolume /]# cat /tmp/apikey/apikey.txt
A19fh68B001j
[root@pod-secretvolume /]# exit
```

## Secret as an environment variable in the Container

This example shows how to inject a secret as a container environment variable.

```console
$ oc apply -f pod-secretenv.yaml 
pod/pod-secretenv created
```

This pod declaration contains the following snippet, which take a secret by name and inject its value within the container into the Pod.

```yaml
...
  containers:
  - name: shell
    image: centos:7
    env:
      - name: SECRET_APIKEY
        valueFrom:
          secretKeyRef:
            name: apikey
            key: apikey.txt
...
```

Now if we now exec into the container you can see your secret injected as an environment variable:

```console
$ oc exec -it pod-secretenv -c shell -- bash
[root@pod-secretvolume /]# env | grep APIKEY
SECRET_APIKEY=A19fh68B001j
[root@pod-secretvolume /]# exit
```


You can remove the pods and the secret with:

```console
$  oc delete pod/pod-secretenv pod/pod-secretvolume secret/apikey 
pod "pod-secretenv" deleted
pod "pod-secretvolume" deleted
secret "apikey" deleted
```
