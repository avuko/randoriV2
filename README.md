# Randori V2

![](randori.gif)

I'm going to be dusting of the old randori (https://github.com/avuko/randori), add a couple of techniques and make it all easy to deploy, manage, and destroy with ansible (https://www.ansible.com/).

## Ansible setup

### DigitalOcean specifics

You can skip all of this if you don't use *DO*.

First I'm going to be using doctl, because my targets are DigitalOcean droplets.

https://github.com/digitalocean/doctl/releases I'm getting the latest tar.gz, unpacking and dropping in my bin

```bash
cd ~
wget https://github.com/digitalocean/doctl/releases/download/v1.52.0/doctl-1.52.0-linux-amd64.tar.gz
tar -xvzf doctl-1.52.0-linux-amd64.tar.gz
mv doctl ~/bin/
chmod u+x ~/bin/doctl
rm doctl-1.52.0-linux-amd64.tar.gz

avuko@zafu:~$ doctl 
doctl is a command line interface (CLI) for the DigitalOcean API.

Usage:
  doctl [command]

Available Commands:
  1-click         Display commands that pertain to 1-click applications
  account         Display commands that retrieve account details
  apps            Display commands for working with apps
  auth            Display commands for authenticating doctl with an account
  balance         Display commands for retrieving your account balance
  billing-history Display commands for retrieving your billing history
  completion      Modify your shell so doctl commands autocomplete with TAB
  compute         Display commands that manage infrastructure
  databases       Display commands that manage databases
  help            Help about any command
  invoice         Display commands for retrieving invoices for your account
  kubernetes      Displays commands to manage Kubernetes clusters and configurations
  projects        Manage projects and assign resources to them
  registry        Display commands for working with container registries
  version         Show the current version
  vpcs            Display commands that manage VPCs

Flags:
  -t, --access-token string   API V2 access token
  -u, --api-url string        Override default API endpoint
  -c, --config string         Specify a custom config file (default "/home/avuko/.config/doctl/config.yaml")
      --context string        Specify a custom authentication context name
  -h, --help                  help for doctl
  -o, --output string         Desired output format [text|json] (default "text")
      --trace                 Show a log of network activity while performing a command
  -v, --verbose               Enable verbose output

Use "doctl [command] --help" for more information about a command.
```

Next, go to https://cloud.digitalocean.com/account/api/tokens to get yourself a token. 

```bash
doctl auth init
```

Add the token and your are (supposedly) done.

Next, go to https://www.digitalocean.com/community/tools/do-ansible-inventory. It will point you to https://github.com/do-community/do-ansible-inventory/releases. 

```bash
wget
https://github.com/do-community/do-ansible-inventory/releases/download/v1.0.0/do-ansible-inventory_1.0.0_linux_x86_64.tar.gz
tar -xvzf do-ansible-inventory_1.0.0_linux_x86_64.tar.gz
mv do-ansible-inventory ~/bin/
rm README.md
rm LICENSE.txt
```

To create an ansible inventory file, I run `do-ansible-inventory --out=inventory`. 

An inventory file is just a plain text file, that looks somewhat like this:

```
randori01	ansible_host=198.51.100.3

[ams1]

[ams2]

[ams3]
randori01

[blr1]

[....SNIP....]

[randoriv2]
randori01
```



