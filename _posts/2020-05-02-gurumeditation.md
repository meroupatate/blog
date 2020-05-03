---
layout: post
title: A possible reason for your Virtualbox VM entering the "gurumeditation" state
tags: virtualbox gurumeditation
---

If you are trying to deploy a Virtualbox virtual machine with Vagrant but your VM entered the "gurumeditation" state, then maybe this article can help you.

I ran into this unexpected error while trying to run `vagrant up` the other day:

> The guest machine entered an invalid state while waiting for it
to boot. Valid states are 'starting, running'. The machine is in the
'gurumeditation' state. Please verify everything is configured
properly and try again.
>
> If the provider you're using has a GUI that comes with it,
it is often helpful to open that and watch the machine, since the
GUI often has more helpful error messages than Vagrant can retrieve.
For example, if you're using VirtualBox, run `vagrant up` while the
VirtualBox GUI is open.
>
> The primary issue for this error is that the provider you're using
is not properly configured. This is very rarely a Vagrant issue.

While my first thought was that I actually messed up something with my Virtualbox configuration as suggested in this log, I quickly ran out of ideas once I had gone thought multiple forums that made me reinstall Virtualbox time after time in different versions without success...

I then proceeded to look into the Virtualbox logs but I found nothing that really helped me to understand what the issue was.

Finally, I found what I was looking for in this [Github issue about minikube](https://github.com/kubernetes/minikube/issues/4913#issuecomment-543560593) !

In fact, it turns out that the problem was that I had another VM existing on my system that was using KVM. It seems that there is some sort of conflict between Virtualbox and KVM that prevents Virtualbox VM to boot if a KVM machine is running.

If you actually need to keep your KVM machine running, you may want to look into other providers for your Vagrant VM such as using Libvirt instead of Virtualbox.

In my case my other VM was started by Minikube using the KVM driver. And since I didn't need it anymore, I simply deleted it by running `minikube delete`.

Then I aborted the gurumeditating VM by running `killall -9 VBoxHeadless && vagrant destroy`, which shouldn't cause too much trouble since there shouldn't be other Virtualbox VM.

After that, running `vagrant up` successfully created and configured the VM I wanted to start.
