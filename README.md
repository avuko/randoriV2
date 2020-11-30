# Randori V2

![](randori.gif)

I'm going to be dusting off the old randori (https://github.com/avuko/randori), add a couple of techniques and make it all easy to deploy, manage, and destroy with Ansible (https://www.ansible.com/).

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



To test if it works (add additional `-vvvv` to `ansible-playbook` to check why it doesn't):

`ping.yml`

```yaml
---

- hosts: randoriv2
  remote_user: root 
  gather_facts: false
  tasks:
    - ping:

```



```bash
ansible-playbook --private-key ~/.ssh/digitalocean -i ansible/inventory ansible/ping.yml 

PLAY [randoriv2] ************************************************************************

TASK [ping] *****************************************************************************
ok: [randori01]

PLAY RECAP ******************************************************************************
randori01 : ok=1  changed=0  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0   
```

## Intitial configuration

Update, upgrade and reboot if required (I'm using Ubuntu systems, YMMV!):

`ansible-playbook --private-key ~/.ssh/digitalocean -i ansible/inventory ansible/up*`

[ansible/update.yml](ansible/update.yml)

[ansible/upgrade_all.yml](ansible/upgrade_all.yml) (named `_all` so wildcard usage runs them in right order)

[ansible/upgrade_reboot.yml](ansible/upgrade_reboot.yml)

## Setting root password

`PasswordAuthentication yes` should be in `/etc/ssh/sshd_config`, otherwise well-behaving clients would not try to use passwords as it is not supported.  This will be done by pushing a configuration later on, but for now you can do this manually if you want to test it. To test what you've done, you can log in like this: 

`ssh -o 'PasswordAuthentication yes' -o 'PubkeyAuthentication no' root@randori01`

The use of `ansible-vault` is less intuitive than I hoped.

These are the steps:

1) **create a file `secrets.yml` with one line**:

`root_password: <your root password>'`

**Then run:** 

`ansible-vault encrypt secrets.yml`

In this way the `set_rootpassword.yml` will prompt for the vault password and set the remote password.

It will also change sshd_config to allow password authentication, otherwise this exercise does not get us a last ditch access to our box. 

```shell
ansible-playbook --ask-vault-pass --private-key ~/.ssh/digitalocean -i ansible/inventory ansible/set_rootpassword.yml
```

```yaml
ansible/set_rootpassword.yml 
---

- hosts: randoriv2
  vars_files:
    - secrets.yml
  remote_user: root
  gather_facts: false
  tasks:
    - user: name=root password="{{ root_password | string | password_hash('sha512') }}"
```

## Tweaking limits.conf

The `limits.conf` needs to be set because, in order to both accept and connect back to a large number of brute-force attacks, we are going to spin up a lot of processes/files. So, we increase it with `set_limits.conf`.

`ansible-playbook --private-key ~/.ssh/digitalocean -i ansible/inventory ansible/set_limits.yml`



## I'm building a script to run all playbooks (in order)

`ansible-playbook --ask-vault-pass --private-key ~/.ssh/digitalocean -i ansible/inventory ansible/run_all.yml`

## NOTES

