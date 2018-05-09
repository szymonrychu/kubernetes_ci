# From 0 to CI/CD Kubernetes way (k8s the intermediate way)

## Dependencies

As a step 0 we need to make sure, that we have all the tools we need to proceed with the tutorial.

There are only few. Let's start with `Vagrant`. It will be needed for setting up cluster locally- it automated vm management nicely and it's an open-source tool vastly used in the community, so Vagrantfiles are popping out everywhere. It's very nice tool to have installed locally.

Next is `Ansible`. In the next few steps we will use `Ansible` to prepare our cluster so we will be able to use it locally.
`Ansible` is a configuration management tool automating various OS tasks that Administrator would have to do manually (in 90').

To actually proceed with the installation we will need to install a python library named `netaddr`. Please make sure that you are installing `netaddr` with the same `pip` you were using to install `Ansible`.

We will also need `kubectl` installed to run API queries in our cluster from host machine.

## Setting k8s cluster with kubespray

Kubespray is a repository containing Ansible scripts installing kubernetes on machines.
It's very nicely automated way to install and scale kubernetes in a cloud, but also in bare metal.

Kubespray also comes with very nice Vagrantfile that enables developer to standup their own cluster locally and this is what we will do in the first step.

First of all you have to clone the repository somewhere:
```
git clone https://github.com/kubernetes-incubator/kubespray.git
```

Let's then bootstrap our server:
```
cd kubespray
vagrant up
```

Vagrant will now handle vm managament, set up proper CPU and memory quotas and prepare you 3 VMs: one kubernetes master and 2 kubernetes workers.

We won't be going too deeply into kubernetes cluster infrastracture now, but in the future kubespray gives very easy ability to add or remove kubernetes workers in a matter of few easy, repeatable commmands.

As long as there is at least one worker running we are good to go.

## Setting up Admin user, getting dashboard permissions

Let's now ssh to our kubernetes machine and prepare admin user for us to view the dashboard.

In the same directory as in previous step, type:
```
vagrant ssh k8s-01
```
This will ssh to the first Kubernetes machine in the cluster, the master one.
Now there is time to create files defining us the admin user, who will give us access to kubernetes dashboard.

Prepare 2 files named `admin_account.yaml`:
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin
  namespace: kube-system

```
and `admin_role.yaml`:
```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin
  namespace: kube-system

```
(filenames doesn't matter as long as you remember what is what ;) ) defining username (`admin`) and namespace (`kube-system`) the account have access to.

Now let's use these files to create the exactl user:
```
kubectl create -f admin_account.yaml
kubectl apply -f admin_role.yaml
```

Last thing that's missing for us to login to the dashboard is obtaining `admin`'s token.

This can be done with one command (in fact two in one-liner):
```
kubectl -n kube-system describe secret admin $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

The value hidden in the `token` field is what you are searching for. Now we can login to kubernetes dashboard.

Let's obtain kubernetes master ip with a command `ip ad sh`- for me it's `172.17.8.101` and go to page `https://172.17.8.101:6443/ui`. Use the token to login to the dashboard.

## Getting host machine's `kubectl` to work

Now let's make our local `kubectl` working. Command uses something called context'es to manage different credentials to different kubernetes clusters.

Let's create us a context, so we will be able to use it in next few steps.

First let's tell kubernetes that there is a new user in the neighbourhood.
We will use the token obtained in the previous step to authenticate in the cluster.
```
kubectl config set-credentials admin --token <token>
```

Then let's tell `kubectl` about the cluster we created. We will use the ip we obtained before:
```
kubectl config set-cluster kubespray --server=https://<cluster_master_ip>:6443 --insecure-skip-tls-verify
```

And let's collect those two informations for `kubectl` into context and let's switch to it. In our case context will be named `kubespray-admin`:
```
kubectl config set-context kubespray-admin --cluster=kubespray --namespace=kube-system --user=admin
kubectl config use-context kubespray-admin
```
In the end we can test if the command is working by typing:
```
kubectl describe all
```
If you can see an output without errors and showing informations like this you *have working kubernetes cluster locally!*:
```
Name:              kubernetes
Namespace:         default
Labels:            component=apiserver
                   provider=kubernetes
Annotations:       <none>
Selector:          <none>
Type:              ClusterIP
IP:                10.233.0.1
Port:              https  443/TCP
TargetPort:        6443/TCP
Endpoints:         172.17.8.101:6443
Session Affinity:  ClientIP
Events:            <none>
```

Disclaimer- why not `minikube`? With minikube I hit several problems and it was not that easy to stand it up in the mac-os.

## Getting jenkins to work

Once we are done with initial setup of cluster and tools we will use, there is time for the fun part.

Let's make jenkins working on the cluster!

Since we have proper context configured in our `kubectl` we don't need to ssh to cluster master to get things moving.

First of all we will start with `jenkins_account.yaml` and `jenkins_role.yaml` files, which contents should look like this:
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: default
```
and this:
```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: jenkins
  namespace: default
```

Let's create and apply the files, so our `jenkins` user with access to `default` namespace will be ready:
```
kubectl create -f jenkins_account.yaml
kubectl apply -f jenkins_role.yaml
```
Now let's obtain its bearer token- which will be needed in a minute.
```
kubectl describe secret jenkins $(kubectl get secret | grep admin-user | awk '{print $1}')
```

Next thing we will be doing is to apply jenkins deployment and create jenkins service.

Let's start with the files- `jenkins_deployment.yaml`:
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  minReadySeconds: 5
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      containers:
        - name: jenkins
          image: szymonrychu/k8s-ros-jenkins:1.0
          env:
            - name: JAVA_OPTS
              value: -Djenkins.install.runSetupWizard=false
          ports:
            - name: http-port
              containerPort: 8080
            - name: jnlp-port
              containerPort: 50000
          volumeMounts:
            - name: jenkins-home
              mountPath: /var/jenkins_home
      volumes:
        - name: jenkins-home
          emptyDir: {}
```

and `jenkins_service.yaml`:
```
apiVersion: v1
kind: Service
metadata:
  name: jenkins
spec:
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: jenkins
```

And let's apply/create these to login to jenkins:
```
kubectl apply -f jenkins_deployment.yaml
kubectl create -f jenkins_service.yaml
```

The files will tell kubernetes, that it should take the `szymonrychu/k8s-ros-jenkins` docker from dockerhub and deploy it as a `jenkins` pod in the cluster.

We have to wait a while for kubernetes to download jenkins'es container:

<img src="https://github.com/szymonrychu/kubernetes_ci/raw/master/images/jenkins_deployment_1.png" width="1024" height="auto">

Once it's done we can get jenkins'es port. You can find it in the `Services` category in the overview:

<img src="https://github.com/szymonrychu/kubernetes_ci/raw/master/images/jenkins_deployment_2.png" width="1024" height="auto">

Or you can get it from terminal with kubectl:
```
kubectl get service | grep jenkins
```
The output will look like this:
```
jenkins      NodePort    10.233.15.84   <none>        8080:32241/TCP   28m
```
And we are interrested in port configuration- *32241* port is what we are looking for. Let's combine cluster ip and newly obtained port and open up jenkins `http://<cluster_ip>:<jenkins_pod_port>/`!

My example: `http://172.17.8.101:32241/`

## Jenkins Kubernetes plugin configuration

Let's open up jenins system configuration and add kubernetes cloud.

Let's go to `Manage Jenkins` -> `Config System` scroll all the way to the bottom to `Cloud` section, hit `Add a new cloud` button and choose `Kubernetes`.

Let's fill up kubernetes connectivity options:

option | value
---|---
Kubernetes URL | https://kubernetes
Disable https certificate check | true
Kubernetes Namespace | default

Let's then create credentials using previously obtained jenkins user's bearer token.

In `Credentials` field, lets hit `Add` and `Jenkins` buttons. Change the `Kind` to `Secret text`, and let's fill up the fields as follows:

<img src="https://github.com/szymonrychu/kubernetes_ci/raw/master/images/jenkins_plugin_configuration_1.png" width="1024" height="auto">

And hit `Add` button. Then let's choose freshly created credentials as the ones we will use to contact kubernetes cluster and let's test the connectivity with `Test Connection` button.

Then let's fill up jenkins address. To obtain internal jenkins'es pod addres you can go to `Pods` -< `jenkins-xxxxxxxxx-xxxxx` and check `IP` of the node.
Otjer way to obtain the address is to use kubectl as follows:
```
kubectl describe pod  $(kubectl get pods | grep jenkins | awk '{print $1}')
```
And get the ip from there.

The port is static thing, and it should be set to `8080`.


option | value
--- | ---
Jenkins URL | http://<jenkins_internal_ip>:8080

Last thing to set-up is to specify what docker container should be used as a workers in our environment.

Let's hit `Add Pod Template` and `Kubernetes Pod Template`.

New Section will appear- let's fill up all necessary options:

option | value
--- | ---
Name | jenkins-worker
Namespace | default
Labels | jenkins-worker

Then let's add source of the container to use for the builds- let's hit `Add Container` and `Container Template`.

In the fields let's put:

option | value
--- | ---
Name | jnlp
Docker Image | jenkinsci/jnlp-slave
Always pull image | yes
Command to run | jenkins-slave
Arguments to pass to the command |

Diclaimer: the `jenkins/jnlp-slave` docker image is a default one used for the builds- if you want to customize it and use your own Docker container instead, **you have to set the Name in previous sections to `jnlp`** (it's written in the plugin docs).

Complete configuration screenshots working for me:

<img src="https://github.com/szymonrychu/kubernetes_ci/raw/master/images/jenkins_plugin_configuration_2.png" width="1024" height="auto">

<img src="https://github.com/szymonrychu/kubernetes_ci/raw/master/images/jenkins_plugin_configuration_3.png" width="1024" height="auto">

## Kubernetes worker job

We have all prepared to run actual commands on the dynamically created worker nodes.

Let's just create sample job, that will use the configuration and prove that everything is working properly!

Hit `New Item` set a name of your choice (my example is `kubernetes-worker-test-job`), choose `Freestyle project` and hit `Ok`.

In `Restrict where this project can be run` -> `Label Expression` put previously specified label of the workers the project should be run on, which in our case is `jenkins-worker`.

Go to the bottom of the screen and hit `Add build step` and `Execute shell`.

Tio prove the concept we just need some long living task to execute in a job, so in the new `Execute shell` field I'm putting just `sleep 30` command.

Let's save such job and fire it up (with `Build now` button)!

After a short while you'll see, that jenkins registers new worker:

<img src="https://github.com/szymonrychu/kubernetes_ci/raw/master/images/jenkins_tutorial_demo_1.png" width="400" height="auto">

<img src="https://github.com/szymonrychu/kubernetes_ci/raw/master/images/jenkins_tutorial_demo_2.png" width="1024" height="auto">

After next few moments, when jenkinsnode get's initialized and the job is being run on the worker.

<img src="https://github.com/szymonrychu/kubernetes_ci/raw/master/images/jenkins_tutorial_demo_3.png" width="400" height="auto">

<img src="https://github.com/szymonrychu/kubernetes_ci/raw/master/images/jenkins_tutorial_demo_4.png" width="1024" height="auto">

After the job is done, container get's disposed.

<img src="https://github.com/szymonrychu/kubernetes_ci/raw/master/images/jenkins_tutorial_demo_5.png" width="400" height="auto">
