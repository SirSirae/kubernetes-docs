# Kubernetes Documentation

## Disclaimer before we start to go deeper into this rabbit hole

The first time I started looking into virtualization, docker, kubernetes and etc... it was for work... a year ago. 

I happen to like it, to the point where I now managed to build a nice kubernetes cluster at home, which can always be improved (and will be improved).
 
If I say all that is because I want one thing to be very clear the main goal of this document is to help you build your homelab, and nothing more I won't go into 
details on how to setup a full blown production ready cluster... That said we won't be very far from that and it will be more than enough for all your homelab needs. 

Second disclaimer english isn't my main language so if there is any mistake let me know.

Third disclaimer, this tutorial is made for linux and mac based environment on windows some commands might differ.
 
Last disclaimer, I consider here that you already have basic knowledge in IT, virtualization and containers so I take some shortcuts here and there.

## So where do we start ? - VMs Setup !

Disclaimer I could use ansible a
First things first we need machines, hardware, something that we can use, possibly destroy, mess with etc you get the idea.
For this purpose I want to use virtual machines as it is easy to recover from an issue and avoid mistakes when we are learning. 
The advantage here is that once we know this setup works on a set of virtual machines we can just replicate this setup on real machines and it should be fine.

There is a TON of ways to setup virtual machines, to just name a few virtual box, vmware, proxmox, terraform are all names you should be familiar with.
In 2020 one would setup everything with terraform and ansible as the tech is exactly made for what we are about to do, however in order not to make things more complicated for beginners I won't use it here for now. 
Feel free to setup your environment this way if you know how to do it, for everyone else let's continue.

So for this demo I will use vmware and setup everything manually, feel free to use virtualbox as it is completely free and does the exact same job, I just use vmware as I have a licence for it.

The first step here is to create three machines, completely identical (you can decide if you want more ram or cpu than what I show here it is up to you).
Those three machines will be run our cluster. 

Each machine I created is setup like this:
- OS Ubuntu server available at https://ubuntu.com/download/server (choose the manual install option and download the ISO, then use it to create your VM)
- 2 CPU Cores
- 4GB of ram
- 20GB of disk

It is clearly overkill for what we are doing, I have the hardware to handle it so there is no issue here, but your mileage may vary. Feel free to adjust each machine to your personal computer hardware.
Also remember we will be running the three machines at the same time so make sure your computer can handle everything before starting.

Now you can install ubuntu server on each machine, it will ask you a bunch of questions about the machines just make sure to install openSSH during the installation process, so we can remote access the machine. 
This step is very important and necessary so pay attention to it when the software asks you ! AND SAY YES !

You can name each machine as you wish, just remember that one machine will be the master and the other two will be the slaves.

I names my machines like this:
- KubernetesMaster
- KubernetesSlave1
- KubernetesSlave2

Each machine will be given an IP by your virtualization platform of choice just make sure to take note of these IPs as they will be necessary in a minute.

At this point each machine should be up and running and you should see a nice login screen (by nice I mean literally just the word login but that's to be expected)

Now what we want to do is add a user that will be responsible to handle the cluster setup, for demonstration purposes we will create a user called k3 (spoiler alert if you are familiar with kube you know where this is going ;) )

In order to do so we must login into each machine and create a user. So we don't handle each machine in the virtualization interface I recommend you to get a terminal (command line on windows) 
which will make our life much easier. On mac I use iTerm2 but feel free to use what ever you want.

Now we have to remote access each machine from our terminal (here is where SSH and IPs comes into play):
```shell script
ssh <username>@<machine-ip>
```
(Of course replace the values marked as <XXX> with your own values)

Side note: to terminate an ssh connection you must type 'exit' into the terminal.

In order to create the user k3 we must do the following command (on each machine):
```shell script
sudo adduser k3
```
This command will ask you your password and then will ask you the k3 user password, I've set mine to 'test' but you can choose whatever you want. 

Now we must allow this user to have admin rights when necessary without having to type the password all the time (we won't have this for very long just for the initial setup, as it saves us some time). 
In order to do so you must do the following:
```shell script
sudo visudo
```

You are now taken to a very important file (do not erase anything here), this file will allow k3 to be able to do sudo commands without password. 
What you want to do now is go to the very end and add the following text (again on all machines):
```shell script
k3 ALL=(ALL) NOPASSWD:ALL
```

One last thing we must do, is to be able to ssh into our machines without having to type k3's password all the time when we connect to it via ssh.
In order to do so, you must exit the ssh connection, by just typing exit in the terminal, this will kill the connection to that machine and bring you back to your original computer.
From your computer you can now do (again three times) the following command:
```shell script
ssh-copy-id k3@<machine-ip>
```
This will ask you some question and a password (k3's password), once everything goes well you can ssh into any of the three machines (as k3) without having to type k3's password. 

OK ! So now we are at the point where things are ready ! We have three machines, all setup with a k3 account that can do pretty much everything on the machines (I know security wise it's not the best but those permissions will be revoked after the setup).

## Installing kubectl

Kubectl is a tool that is used to access and run commands on a kubernetes cluster, it's primordial that you have this tool as it is THE tool to do pretty much everything on your cluster.
You can follow this link in order to install the tool on your machine: https://kubernetes.io/docs/tasks/tools/install-kubectl/

## Installing kubernetes on our machines.

The kubernetes flavour we will setup today is K3S, there is tons of other version out there but I found out after an extensive research that K3S has many advantages, first of all it is very easy to install, has a very small footprint and is backed up by rancher 
which is a big company in the kubernetes space. All that aside we also are lucky cause someone already made a pretty cool tool for us to use in order to install kubernetes with a very simple command.

The tool we will use for the job is k3sup (https://github.com/alexellis/k3sup) I invited you to check his readme but if you don't have time I will put here the important parts.

The first step is the installation on our computer (not on the VMs) of the k3sup tool to do so you need the following command:
```shell script
curl -sLS https://get.k3sup.dev | sh
sudo install k3sup /usr/local/bin/
```

Once this is done we can now proceed and install k3s on one of our virtual machines, this machine will become the master so choose carefully.
In order to install k3s you must run the following command:
```shell script
k3sup install --ip <master-server-ip> --user k3 --k3s-extra-args '--no-deploy traefik'
```
Side note: as you can see here I do --k3s-extra-args '--no-deploy traefik' this is because I do not want to install the traefik that comes with k3s as it is version 1.7, and we are now at 2.3.X, which brings to the table bunch of amazing feature that 1.7 lacks.

You should have a message that lets you know that all is setup correctly and some commands written for you to test it out.
Basically it tells you to export the kubeconfig file to a variable and then do 
```shell script
kubectl get nodes
```
Here you should see the node you just installed.

Now we must add the two other machines to our cluster so that we have a nice multiple node cluster running our things.
K3sup comes in handy once again as there is a simple command to do just that:
```shell script
k3sup join --ip <machine-to-add-ip> --server-ip <master-server-ip> --user k3
```
Do this command to both new machines that you want to add to your cluster.

Feel free to add as many machines as necessary I just did three in this example but technically you can go as high as your hardware allows you to.

## Now onto installing stuff to make everything work

(Side note: I won't use helm in this tutorial as I want to be able to understand and control everything that is happening. This approach helped me understand kubernetes instead of relying on the magic helm brings, I believe while learning this approach is better
and once you fully understand what is happening feel free to use helm.)

We are almost at the end of this small doc (I imagine cause I still didn't write everything, if it continues for our don't blame me please)... Now we have a running cluster fully ready to take one what you want to put on it.

But we just need to add a couple of stuff for everything to be 100% ready.

(Be ready, here I will start talking in kubernetes terms if you don't understand those terms you will have to look into it in the official kubernetes documentation: https://kubernetes.io/)

The first thing we must have is called metallb, this tool allows you to create load balancer service on your cluster. Which will be relevant later when we tackle other software.

In order to install metallb there is a couple of commands we must use (versions may vary depending on when you are reading this feel free to change it in the links below if a newer version is available):
```shell script
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.4/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.4/manifests/metallb.yaml
# On first install only
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```
Those files will create everything in order to use metallb in your cluster, EXCEPT one thing the configuration for it which you must do yourself.
In order to make this configuration you must create a yaml file, with your text editor of choice, let's call it 'ips-configmap.yml'.
Here is my configuration just remember to add your ip range to the file and change the name of the address-pool (those IPs will be attributed by metallb to your service so you should use IPs you know you can reach, in the case of your home network if your home is 192.164.1.X,
you can for example give a range like 192.164.1.200-192.164.1.250, if you stay within this tutorial look at the ips your VMs have, if your VMs have for example ips like 123.123.123.12 you can do a range with 123.123.123.XXX - 123.123.123.YYY):
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: <default-name>
      protocol: layer2
      addresses:
      - <start-ip>-<end-ip>
``` 

Save the configuration file and apply it to the cluster (to do the following command you must be in the same folder as your yml file otherwise you must add the path to it like /path/to/config/ips-configmap.yml):
```shell script
kubectl apply -f ips-configmap.yml
```
If it says that the configmap was created you are golden ;)

## On to reverse proxy and ssl certs

The solution we will use for this is called Traefik, it is pretty much bullet proof, used in the industry by tons of companies and free, which makes it in my honest opinion the best tool for the job, once you understand the tool and once it is setup
you can forget about it, it just work.

So onto traefik. To do so we will need to add some configurations to our cluster. Those are objects that traefik understands and needs in order to function properly.
Here is my file you can copy paste it, let's name this file 'ingressRouteDefinition.yml':
```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressroutes.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRoute
    plural: ingressroutes
    singular: ingressroute
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: middlewares.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: Middleware
    plural: middlewares
    singular: middleware
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressroutetcps.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRouteTCP
    plural: ingressroutetcps
    singular: ingressroutetcp
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressrouteudps.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRouteUDP
    plural: ingressrouteudps
    singular: ingressrouteudp
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: tlsoptions.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TLSOption
    plural: tlsoptions
    singular: tlsoption
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: tlsstores.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TLSStore
    plural: tlsstores
    singular: tlsstore
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: traefikservices.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TraefikService
    plural: traefikservices
    singular: traefikservice
  scope: Namespaced

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller

rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
      - networking.k8s.io
    resources:
      - ingresses
      - ingressclasses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses/status
    verbs:
      - update
  - apiGroups:
      - traefik.containo.us
    resources:
      - middlewares
      - ingressroutes
      - traefikservices
      - ingressroutetcps
      - ingressrouteudps
      - tlsoptions
      - tlsstores
    verbs:
      - get
      - list
      - watch

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller

roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
  - kind: ServiceAccount
    name: traefik-ingress-controller
    namespace: default
```
As usual we must now apply this to the cluster, same as before:
```shell script
kubectl apply -f ingressRouteDefinition.yml
```

Here you will see that bunch of stuff have been created, that's normal and to be expected.

Now we must create a deployment and a service account for our traefik for it to work, you can do soo by applying again another yaml file (everything is yaml in kube)
Here is the file just make sure to check the comments and change according to your needs, also note that I am installing traefik in the default namespace you can change it if you want make sure to create a namespace beforehand.
Let's call this one traefik.yml:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: default
  name: traefik-ingress-controller

---
kind: Deployment
apiVersion: apps/v1
metadata:
  namespace: default
  name: traefik
  labels:
    app: traefik

spec:
  replicas: 1
  selector:
    matchLabels:
      app: traefik
  template:
    metadata:
      labels:
        app: traefik
    spec:
      serviceAccountName: traefik-ingress-controller
      containers:
        - name: traefik
          image: traefik:v2.3 # you might have a higher version by the time you reading this
          args:
            # This should not be setup when you are in production as it creates a dashboard that can be accessed by anyone
            # but for our test needs it is great just remember to remove it 
            - --insecure.api
            - --accesslog

            # Here we define our entry points we have two of them one at 80 (we call it web) and one at 443 (we call it websecure)
            - --entrypoints.web.Address=:80
            # Traefik handles automatic redirections from http to https and it's done like so
            # feel free to comment that in the first time if you want to test your http endpoint
            - --entrypoints.web.http.redirections.entryPoint.to=web-secure
            - --entrypoints.web.http.redirections.entryPoint.scheme=https
            - --entrypoints.web.http.redirections.entrypoint.permanent=true
            - --entrypoints.web-secure.Address=:443
            
            # I still need to read about providers but basically we need that 
            - --providers.kubernetescrd

            # This part is the part that will generate our ssl certificates
            # I invite you to read a bit more about this, you will need your own domain name in order to use it
            # Traefik has a nice documentation on the different options but in the meantime here is what I used
            - --certificatesresolvers.certresolver.acme.tlschallenge # many challenges exists you must see what you prefer/need
            - --certificatesresolvers.certresolver.acme.email=your@email.com # replace this with your mail
            
            # This file will store our certificates, I do not use a volume to store this so everytime traefik reboot it will destroyed
            # this is fine for our dev purposes but you might want to have a volume when you go live with your cluster 
            # it is important for you to know that we will use letsencrypt and it has restrictions on the amount of certificates we can ask for
            # if you ask more then X certificate for test.domain.com you will be throttled and will have to wait to have your certificate
            # here we are going to use the staging server so we can make sure the certificate validation works and then we will remove the staging and go for real certs
            - --certificatesresolvers.certresolver.acme.storage=acme.json 
            # here we setup who will give us certificates we chose the staging server as explained above as we want to make sure it works first
            # to have real certificate remove staging from the link below
            - --certificatesresolvers.certresolver.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory
          # here we are opening ports in order to access later traefik
          # we have the usual 80 and 443 but also 8080 this last one is where the traefik dashboard is deployed
          # for testing it is ok but remember we set this as insecure so in the future we must protect this or simple not make it acessible
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
            - name: admin
              containerPort: 8080
```

And once again we must apply that:
```shell script
kubectl apply -f traefik.yml
```

Now traefik should be running (validate it with kubectl get deployment to make sure) but it is not yet accessible to you, it's still trapped inside the cluster.
In order to access it we must have a service (of type loadbalancer here is where metallb comes in handy).
Here is the service file let's call it service.yml:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: traefik
  annotations:
    metallb.universe.tf/address-pool: <name-of-your-ip-pool-defined-above>
spec:
  ports:
  - port: 80
    targetPort: 80
    name: http
  - port: 443
    targetPort: 443
    name: https
  - port: 8080
    targetPort: 8080
    name: admin
  selector:
    app: traefik
  type: LoadBalancer
```

You know the drill we must apply the little fellow:
```shell script
kubectl apply -f service.yml
```

Now the magic is almost about to happen !

## Let's put it to the test

Ok now we have a setup where we can in fact create our certificates for our services and so on, in order to keep it simple however we will use an image that is already available to us.
We will setup now nginx (you don't really need to know what it is) on our custom domain and validate the certificate.

Bunch of things we must do first though.
Let say we want to expose test.example.com we must say to our computer to redirect test.example.com to the virtual machine (the entry point of our cluster), in our case it is the service we just created for traefik !
By running this command:
```shell script
kubectl get service
```
You should see a line like so:
```shell script
traefik          LoadBalancer   10.43.79.253   <the-ip-we-want>   80:31465/TCP,443:32256/TCP   21h
```
Notice the second IP this is the IP of the service that redirects to traefik, aka what we want.

So now you must edit your host file so that test.domain.com redirects to this IP.
You can check it here: https://www.howtogeek.com/howto/27350/beginner-geek-how-to-edit-your-hosts-file/

Once this is done if you try to access it... you won't see anything, thats expected :D
But you should be able to access <the-ip-we-want>:8080 and see traefik alive if this is ok you are golden !

Now what we must do is add something that answers when we query test.domain.com and not just a blank 404 page.

Here we go installing nginx by doing the following:

First we must create a deployment let's call it ndeploy.yml:
```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  namespace: default
  name: nginx
  labels:
    app: nginx

spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - name: http
              containerPort: 80
```

Apply it with: 
```shell script
kubectl apply -f ndeploy.yml
```

Now we must have a service so that the service can take us to that deployment, let's call it nservice.yml:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service

spec:
  ports:
    - name: http
      port: 80
  selector:
    app: nginx
```

Finally we must tell traefik that we want to access this service when we reach test.domain.com this is done via a route, there is much more information on the traefik website so you can also check it out there if this is complicated for you.
Let's call this the nroute.yml:
```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: nginx-route
  namespace: default
spec:
  entryPoints:
    - http
  routes:
  - match: Host(`test.domain.com`)
    kind: Rule
    services:
    - name: nginx-service
      port: 80

---
# Here we are defining two routes one in http and another one in https
# If you have kept the global http to https redirecting you don't need the http route
# as all traffic will be redirected automatically
# I just left it here for you to see the difference between the two

apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: nginx-route-secure
  namespace: default
spec:
  entryPoints:
    - https
  routes:
  - match: Host(`test.domain.com`)
    kind: Rule
    services:
    - name: nginx-service
      port: 80
  tls:
    certResolver: certresolver
```

What we are saying here is simple, we tell traefik that we want to reach the nginx-service when we arrive with test.domain.com, in http or https

Apply it with: 
```shell script
kubectl apply -f nroute.yml
```

Now if everything has been done correctly you can now open your browser and go to test.domain.com

You will see a warning that's because the certificate will not probably work as you probably don't own test.domain.com and that is normal but you can bypass this warning and reach your site

Now you must just confirm that everything is working and remove the staging server from traefik and replace it with the production server so you can start having your own certificates.

If you don't have a domain name, you can also decide not to implement the certificate resolver by removing the arguments from the traefik deployment, doing so will not resolve certificates and you will have warnings when acessing your site
but as soon as your hosts are configured to reach traefik service correctly within your local domain you can have your own urls for your own apps and everything will run just fine.

Just a tip don't bother with https if you don't have to have certificates, or if your cluster is never opened on the internet, you can run everything on http and have your own urls as well, just remember there won't be encryptions between you and your app if that's what you decide to do.
  
## Let's add NFS storage to our pods !

OK so in this update we will add volumes to our pods so that we can keep data on disk if we restart our pods.
If yu followed along this tutorial of some sort you have now a traefik instance that automatically gets you a certificate once a new route is created.
This is super convenient but once you remove the "staging" let's encrypt server you will see that you might reach a certificate limit, aka you asked too many times for the same certificate.

That happens because as you turn off your traefik instance (for whatever reason) and run it again it will ask again and again for the certificate for the various routes you created.

Reaching your limit very quickly.

In order to avoid that we will create three things (and again we will do so without any automation so we really understand what is happening):

- A volume
- A volume claim
- A traefik deployment with updated parameters

Ok so let's go do that ! BUT before that, I am not entering here into the NFS territory I suppose you already have an NFS server with users that have the right to use a specific volume of your NFS server.
Considering the following for the rest of this tutorial:

- You have an NFS server with a folder in it that you have access to
- You have a user ID and GID at hand and this user has permissions to use the NFS share

In my case I created a volume on my nas and gave a user "kube" the right to access this folder, this kube user has an ID of 1040 and a GID of 200, but of course yours may vary, and thos values are random invented for this tutorial.
I also made my NFS share accept direct connections from my kubernetes IPs and local machine IPs so I can access the NFS share directly without being bothered by access codes.
This vary per NAS distribution so I can't really help you setting this up, but a quick search with "NFS on my XXX nas" should help you.

OK so now we can tackle the kube side of things.

First thing first we will create a volume, this object is the one mapping your NFS share and making it accessible on kubernetes.
In my example below I created a volume called traefik-data-pv and gave it a capacity of 5gigs. I also made sure I have a "readWriteOnce" option which means only one claim can be made on that volume
(more on that later). I also flyover storage class here and use the default, you may want to read more on that but let say here that I don't have different kinds of storage
usually you would use that create a class based on SSD and HDD for instance so that some volumes are automatically stored on SSD, this kind of things. Imagine that as premium storage, basic storage, medium storage and so on. 
Not so relevant for me though.
The interesting part is the "nfs" options where you specify a directory in my NFS volume and the IP of my NFS server. 
```yaml
## PV traefik data

apiVersion: v1
kind: PersistentVolume
metadata:
  name: traefik-data-pv
spec:
  capacity:
    storage: 5Gi
  storageClassName: "local-path"
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  mountOptions:
      - hard
  nfs:
    path: "/vl3/Kubernetes/traefik" # insert here your NFS path where traefik will save data
    server: "XX.XX.XX.XX" # insert here your NFS server IP

```

Now that is done we can create a "claim" an object that will basically use the volume, this claim makes some kind of a bridge between the volume and a kubernetes pod.
Think of it as inserting an usb drive to the pod itself in some way. Note that the claim must match the volume created you can't have a claim that ask for bigger storage then a volume.
You can however have a claim that ask for less storage but remember a volume can only support one claim so better use all the storage.

Here is my claim (also note that it references the previous volume):

```yaml
## PVC traefik data

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: traefik-data-pvc
spec:
  volumeName: traefik-data-pv
  resources:
    requests:
      storage: 5Gi
  storageClassName: "local-path"
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
```

Now as usual we can save thos definition in a file like "volume.yml" and apply that with a good old:
```
kubectl apply -f volume.yml
```

Now the important thing is if you now do:
```
kubectl get pvc
```
You should see your claim with a status "Bound" this means the claim found the volume and the volume is ok !
AKA you are on the good direction.

Now for the final step we want to make sure our traefik is saving the certificates on that volume. Here is an updated version of the traefik deployment file that does just that:
(I removed the previous explanation comments written previously so we can focus on the volume part of things feel free to scroll up for more info on labels)
```yaml
---
kind: Deployment
apiVersion: apps/v1
metadata:
  namespace: default
  name: traefik
  labels:
    app: traefik

spec:
  replicas: 1
  selector:
    matchLabels:
      app: traefik
  template:
    metadata:
      labels:
        app: traefik
    spec:

      # now this security context will vary according to how you have setup things on your nfs share.
      # if you created a share and said "this IP is allowed to do everything" then you don't need the security context
      # if on the other hand you said "no only this user is allowed to do something" then you must tell your pod to act as this user
      # otherwise you will run into security issues such as "no permission allowed" when you will try to save data on the NFS share
      # in my case I have a double safety I say this IP and this account only can use my share so I must specifiy here the account "kube" I created earlier on my nas
      securityContext:
        runAsUser: 1040 # my kube user ID
        runAsGroup: 200 # my kube group ID
      
      serviceAccountName: traefik-ingress-controller
      containers:
        - name: traefik
          image: traefik:v2.3
          args:
            - --accesslog
            - --entrypoints.web.Address=:80
            - --entrypoints.web.http.redirections.entryPoint.to=web-secure
            - --entrypoints.web.http.redirections.entryPoint.scheme=https
            - --entrypoints.web.http.redirections.entrypoint.permanent=true
            - --entrypoints.web-secure.Address=:443
            - --providers.kubernetescrd
            - --certificatesresolvers.certresolver.acme.tlschallenge
            - --certificatesresolvers.certresolver.acme.email=you@email.com
            # Here if I would have left acme.json Kubernetes would have seen this as a folder even if I have a sub path onmy volume mount so I decided to 
            # put the acme.json file in a folder called certs at the root of this container and it works just fine
            - --certificatesresolvers.certresolver.acme.storage=/certs/acme.json 

            # Careful here I removed the staging server and put the real one cause I already know this is working for me 
            # make sure you have staging first so you can test things
            - --certificatesresolvers.certresolver.acme.caserver=https://acme-v02.api.letsencrypt.org/directory
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
          # Here is the interesting part I am here defining a mounting point so that in my cluster the path /certs is actually mounted on the nfs-data volume
          # which I add to my deployment later on
          volumeMounts:
            - name: nfs-data
              mountPath: /certs
      # Here is the volume nfs-data and as you can see I reference here my volume claim that I created earlier.
      # nothing really fancy must be done once you understand it is quite straight forward
      volumes:
        - name: nfs-data
          persistentVolumeClaim:
            claimName: traefik-data-pvc # the name of the claim you created earlier
```

OK so at this point once you "kubectl apply -f" this new traefik deployment you should see it starting up and if your permissions and account are setup correctly you will see your acme.json pop in your NFS folder.

If this is not the case check the logs of traefik as they are rather explicit:
```shell script
kubectl logs -f <traefik pod> 
```

OK so now not only your have a multi node kubernetes cluster but also a reverse proxy with ssl certificates AND nfs storage !
You are pretty much rock solid to start whatever apps you might want !

