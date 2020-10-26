# Kubernetes Documentation

## Disclaimer before we start to go deeper into this rabbit hole

The first time I started looking into virtualization, docker, kubernetes and etc... it was for work... a year ago. 

I happen to like it, to the point where I now managed to build a nice kubernetes cluster at home, which can always be improved (and will be improved).
 
If I say all that is because I want one thing to be very clear the main goal of this document is to help you build your homelab, and nothing more I won't go into 
details on how to setup a full blown production ready cluster... That said we won't be very far from that and it will be more than enough for all your homelab needs. 

Second disclaimer english isn't my main language so if there is any mistake let me know.

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

## Installing kubernetes on our machines.




