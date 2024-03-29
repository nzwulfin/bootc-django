# django-demo

Use ansible to create a Django 5 environment on top of a bootc based OS with the correct python 3 pacakges to support it


The bootc container only contains the OS level mods required for this Django layer to become a functional app environment once configured at run time

The ansible playbooks contain 'generic' customizations that wire up the django environment according to local standards in preparation for landing an application later

On the **build host** you'll need the following packages:
* podman
* ansible
* general, podman, and postgresql ansible collections


```
sudo dnf install -y podman ansible

ansible-galaxy collection install containers.podman
ansible-galaxy collection install community.general
ansible-galaxy collection install community.postgresql
```

We'll also use a local insecure registry to save on network round trips for building later.  You can use any registry container and add it to ```etc/containers/registries.conf``` so it can be used. 

The address should be available on the network if you'd like to test updating a host as the repository is part of the bootc entry.

```
[[registry]]
location = "192.168.122.88:5000"
insecure = true

```

Since we are building 'gold images' that will be installed in several possible ways and configured post boot, we want to leave the container ```machine-id``` alone.  Instead to get the ngingx, postgresql, and gunicorn services enabled in the final images, we use a systemd preset file and add it to the container.  We know what those services are called, so we can add it at build time via the Containerfile.  This will also let us test services in the local container as we go.

Once you have the prerequisites, from the root of this repo you can build and run the container. The name should match the ansible inventory list in the inventory.containers file.

```
podman build -t bootc-django-t1 bootc/
podman run -d --name atest bootc-django-t1 /sbin/init
```

Run the site playbook to make all the config changes to prep this as a deployable django environment ready for an application. The selinux context setting is being worked on, but should get autorelabeled when built by image builder.

```
ansible-playbook -c podman -i ansible/inventory.containers ansible/site.yml
```

Once the playbook completes, we have the container configured for our django standards. There aren't any exposed ports in this config, but you can confirm the skeleton is working from a bash shell. 

```
podman exec -it atest /bin/bash
bash-5.2# systemctl status nginx postgresql gunicorn.socket
bash-5.2# curl --unix-socket /run/gunicorn.sock localhost
```

Convert the configured container to an image, and push it to the local registry to be used for image building.

```
podman stop atest
podman commit atest bootc-django-conf
podman push bootc-django-conf 192.168.122.88:5000/bootc-django-conf
```
**To create an importable QCOW image:**
If needed you can create any new configs needed for the qcow before running the build, like the interactive user.  Create the target directory for the image builder artifacts, then run the image builder container. 

```
mkdir output

sudo podman run --rm -it --privileged --pull=newer --network=host --security-opt label=type:unconfined_t  -v $(pwd)/config.json:/config.json  -v $(pwd)/output:/output  quay.io/centos-bootc/bootc-image-builder:latest --tls-verify=false --type qcow2   --config /config.json 192.168.122.88:5000/bootc-django-conf
```

Get the QCOW image from the output/qcow directory and import into a KVM environment to test.

**To install the image via kickstart:**
If you need to add an interactive user, you can do it in the kickstart file. Push the image you want to install to a secure registry, then reference that URL in the ostreecontainer line.

```
podman push bootc-django-conf quay.io/mmicene/bootctest
```


**To make this more portable:**
There are a few hardcoded IPs that are workarounds for the current playbooks but probably would raise issues in a production setting
