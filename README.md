May 10 2020

Guide to installing CF4K8s on Microk8s on MacOs (should work with minor tweaks on Linux/Windows)
By Neil Isserow
Customer Success Manager - MAPS
May 2020

This is a guide that I created and worked for me. I would be happy to help anyone wanting to get this working. I am not a TAS/PCF expert, this was just a helpful way for me to use TAS on my Mac with k8s. Please also note versions may be updated below so check for latest! This install takes approx 20 minutes or less on my Mac. I also use nano and other tools, please use whatever you find suitable.

I have been asked why Ubuntu and MicroK8s, the answer is Ubuntu is simple and widely known, and MicroK8s is also simple and avoids having to build using Kubeadm and is well supported.

Please note that I will update and also modify as mistakes etc, are found, please feel free to message me. I hope this is helpful for playing around with CF4k8s on your laptop.

BETA NOTE! This software is in BETA and is not to be used in production!

Release Notes: https://github.com/cloudfoundry/cf-for-k8s 

Install Ubuntu 18.04 VM
I simply install on Fusion on my MAC. Take note of the user and pwd you use.

If you use multipass be aware of issues with networking and MetalLB!

Create a new VM I use this for my current Mac spec but you can use larger or maybe smaller (have not tested)
I do a default Ubuntu install from the ISO
Check Microk8s (or install after with sudo snap install microk8s)

ssh to the new Ubuntu Host!

sudo /snap/bin/microk8s.enable dns dashboard metrics-server storage metallb

Choose a IP range when asked for MetalLb that is suitable. I keep it simple and choose the NAT for my VM .200-.240 which is more than enough for testing.

sudo /snap/bin/microk8s.config > kubeconfig

I simply cat this file and copy contents or whatever suits to get it onto your host system

Configure your kubectl config files
For demo I just copy kubeconfig over my current config as I just run 1 at a time but you can append as required for your configuration. I suggest tools like kubens and kubectx if you do this to make it easy to work (brew install kubens kubectx)

Install pre-requisites: (https://github.com/cloudfoundry/cf-for-k8s/blob/master/docs/deploy.md#required-tools)
cf cli (v6.50+)
kapp (v0.21.0+)
ytt (v0.26.0+)
kubectl

Installing CF4K8S
The official documentation can be found here: https://github.com/cloudfoundry/cf-for-k8s 

These were my basic steps:
Get the Git repo - git clone https://github.com/cloudfoundry/cf-for-k8s.git 
cd cf-for-k8s
Create the installation values:
The cf-domain is the wildcard domain you will use. We will set this up using DNSMASQ later but make sure you get this right. I always use a .local for these to ensure no issues with external DNS.
mkdir configuration-values
./hack/generate-values.sh -d <cf-domain> > configuration-values/cf-values.yml
This creates a cf-values file which contains all of the required information for the install. If you do not use the hack you will need to do this manually. Please see official documentation to do this. (https://github.com/cloudfoundry/cf-for-k8s/blob/master/docs/deploy.md#option-b---create-the-install-values-by-hand)

Adding container registry:
I use docker, you can use whatever is currently supported and you choose (At the time of writing Docker, Harbor, ACR).

cd configuration-values
nano cf-values.yml
(copy below into this file at the end)

app_registry:
   hostname: https://index.docker.io/v1/
   repository: "your-docker-hub-login"
   username: "your-docker-hub-login"
   password: "your-docker-hub-pwd"

INSTALLATION:
You should now be ready to install with the following script provided:

From the cf-for-k8s directory:
ytt -f config -f configuration-values/cf-values.yml > configuration-values/cf-for-k8s-rendered.yml
kapp deploy -a cf -f configuration-values/cf-for-k8s-rendered.yml -y

I like to watch so I will execute a watch kubectl get pods --all-namespaces

Once complete continue to DNS and ensure you know the LB IP!

DNS Setup
You will need DNS for your environment, I use DNSMASQ which is very easy to use. The IP is the address for the public router which you will find once install is completed and should be easy to find via kubectl get svc --all-namespaces and it should be the only 1 listed with a loadbalancer provided by our metallb installation config and therefore at 192.168.64.100 *or whatever yours is).

Here are basic instructions:
brew install dnsmasq
cd /usr/local/etc
nano dnsmasq.conf ( you may need to add other items to this file but this is the least)
Add the following to the top (customize as appropriate ie. system.local or mytask8s.local or just .local for all .local):
address=/.local/192.168.64.100
sudo launchctl stop homebrew.mxcl.dnsmasq
sudo launchctl start homebrew.mxcl.dnsmasq
Then test by pinging x.local (or whatever you have set) to ensure all wildcards are pinging the correct address.

Simple CF for testing (yes I am a noob at this and had to write this down :)-)
To test if it worked these are my steps:

cf api --skip-ssl-validation https://api.whateveryouchose.local
cf auth admin PWD-from-configuration-values/cf-values.yml-file
cf create-org test-org
cf target -o "test-org"
cf create-space dev
cf target -o "test-org" -s "dev"

I have a very simple spring boot to test which is basically this! - https://spring.io/quickstart and follow until you have the app running locally successfully via your browser http://localhost:8080/hello

Once you have this cd to the directory and then:
cf push demo

Again this can take some time on my laptop so i will do a watch:
watch cf apps
K9s to check the cf-workloads-staging pods

If you are wondering what will happen here is some info that I have gathered but may need more clarity as well as definition as this is all I can see:
You can watch on your hub.docker.com repositories and a new repository should be created with a GUID (this will be public repository bear that in mind! You can change this to private after however the GUID will change each time and for the free docker hub you only get 1 private repository before having to pay so might want to switch to a local harbor registry)
TAS will do this deploy in steps so could take a while, I use k9s to watch but you will see the following pods/containers that are helpful:
In the first instance you will see the namespace cf-workloads-staging create a guid which you can follow, basically a number of steps as follows (not sure of exact order) - analyze, detect,  build, prepare, restore, export, completion.

Next the namespeace  cf-system creates a pod with guid and containers as follows: istio-init, istio-proxy and cfroutesync. These seem self explanatory and are creating the route/sidecar etc.

Another pod created in cf-system NS for the eirini-guid similar to above

Another pod created in cf-system NS for logs

Finally in the NS cf-workloads a pod with guid demo-guid created


There may be a few more I have missed but if you get this far and have this running with this simple demo spring boot app you should be able to just head to the route in your browser for example: http://demo.myk8s.local/hello and voila it’s working.

-----
