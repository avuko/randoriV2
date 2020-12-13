# Randori V2

![](randori.gif)

I'm going to be dusting off the old randori (https://github.com/avuko/randori), add a couple of techniques and make it all easy to deploy, manage, and destroy with Ansible (https://www.ansible.com/).

## Ansible setup

## TL;DR 

```shell
ansible-playbook --ask-vault-pass --private-key ~/.ssh/do -i ansible/inventory ansible/run_all.yml`
```

There is one playbook to run all the others. Caveat emptor: see  'Setting root password', as you need to create a specific `secrets.yml` file. You obviously also need an `inventory` file and a private key. Group all the hosts you want to set up under the `[randoriv2]` heading in your `inventory`. 

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

## Create inventory

```shell
do-ansible-inventory --out=inventory`
```

To create an ansible inventory file, run the command above. 

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

This is an `INI` style file, but as we are doing everything with yml, I'm going to convert it so it looks like this:

```bash
ansible-inventory -i inventory -y --list > inventory.yml
```

```yaml
all:
  children:
    ams1: {}
    ams2: {}
    ams3:
      hosts:
        randori01:
          ansible_host: 198.51.100.3
    blr1: {}
    fra1: {}
    lon1: {}
    nyc1: {}
    nyc2: {}
    nyc3: {}
    randoriv2:
      hosts:
        randori01: {}
    sfo1: {}
    sfo2: {}
    sfo3: {}
    sgp1: {}
    tor1: {}
    ungrouped: {}

```



To test if it works (add additional `-vvvv` to `ansible-playbook` to check why it doesn't):

`ping.yml`

```yaml
---

- hosts: randoriv2
  vars_files:
  - variables.yml
  tasks:
    - ping:

```



```bash
ansible-playbook --private-key ~/.ssh/do -i ansible/inventory.yml ansible/ping.yml 

PLAY [randoriv2] ************************************************************************

TASK [ping] *****************************************************************************
ok: [randori01]

PLAY RECAP ******************************************************************************
randori01 : ok=1  changed=0  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0   
```

## Initial configuration

```shell
ansible-playbook --private-key ~/.ssh/do -i ansible/inventory ansible/up*`
```

Update, upgrade and reboot if required (I'm using Ubuntu systems, YMMV!)

[ansible/update.yml](ansible/update.yml)

[ansible/upgrade_all.yml](ansible/upgrade_all.yml) (named `_all` so wildcard usage runs them in right order)

[ansible/upgrade_reboot.yml](ansible/upgrade_reboot.yml)

## Setting root password

```shell
ansible-playbook --ask-vault-pass --private-key ~/.ssh/do -i ansible/inventory.yml ansible/set_rootpassword.yml
```

`PasswordAuthentication yes` should be in `/etc/ssh/sshd_config`, otherwise well-behaving clients would not try to use passwords as it is not supported.  This will be done by pushing a configuration later on, but for now you can do this manually if you want to test it. To test what you've done, you can log in like this: 

`ssh -o 'PasswordAuthentication yes' -o 'PubkeyAuthentication no' root@randori01`

The use of `ansible-vault` is less intuitive than I hoped.

These are the steps:

1) **create a file `secrets.yml` with one line**:

`root_password: <your root password>'`

**Then run:** 

`ansible-vault encrypt secrets.yml`

In this way the `set_rootpassword.yml` will prompt for the vault password and set the remote password.

It will also change `sshd_config` to allow password authentication, otherwise this exercise does not get us a last ditch access to our box. 

```yaml
ansible/set_rootpassword.yml 
---

- hosts: randoriv2
  vars_files:
    - secrets.yml
    - variables.yml
  remote_user: root
  gather_facts: false
  tasks:
    - user: name=root password="{{ root_password | string | password_hash('sha512') }}"
```

## Tweaking limits.conf

```shell
ansible-playbook --private-key ~/.ssh/do -i ansible/inventory.yml ansible/set_limits.yml`
```

The `limits.conf` needs to be set because, in order to both accept and connect back to a large number of brute-force attacks, we are going to spin up a lot of processes/files. So, we increase it with `set_limits.conf`.

## The golang environment

```shell
ansible-playbook --private-key ~/.ssh/do -i ansible/inventory.yml ansible/build_essentials.yml`
```

```bash
ansible-playbook --private-key ~/.ssh/do -i ansible/inventory.yml ansible/golang_install.yml
```

Setting up the build environment is necessary because of dependency on packages to compile from source. This also installs things we'll need down the line to get our services running, patch OpenSSH and compile the golang source.. 

## Randori software installation

`ansible-playbook -vv --private-key ~/.ssh/do -i inventory.yml randori_install.yml`

The golang source is old, so the libraries are being replaced, and it all has still to be debugged.
The files are copied over, and when there is a change detected, it's recompiled. Additionally it creates a `/var/log/randorilog` file, which I currently can't remember what its there for.

>  To be continued

## NOTES

Possibly helpful (at least for me) dump of pages I visited, answering the question "A how many tabs problem was this?" and preventing me from hunting through my browser history.

These might or might not be valid/ in existence by the time you read this.

https://docs.ansible.com/ansible/latest/user_guide/vault.html#storing-passwords-in-files

https://www.mydailytutorials.com/ansible-add-line-to-file/

https://stackoverflow.com/questions/24334115/ansible-lineinfile-for-several-lines

https://docs.ansible.com/ansible/latest/collections/ansible/builtin/blockinfile_module.html

https://www.digitalocean.com/community/tutorials/how-to-build-go-from-source-on-ubuntu-16-04

https://stackoverflow.com/questions/56436906/how-to-cleanly-edit-sshd-config-for-basic-security-options-in-an-ansible-playboo

https://stackoverflow.com/questions/62467670/ansible-module-to-stop-and-start-ssh-service

https://docs.ansible.com/ansible/latest/collections/ansible/builtin/file_module.html

https://www.mydailytutorials.com/ansible-create-directory/

https://golang.org/doc/install?download=go1.15.5.linux-arm64.tar.gz

https://docs.ansible.com/ansible/latest/collections/ansible/builtin/get_url_module.html

https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#defining-simple-variables

https://abdennoor.medium.com/setup-go-with-ansible-for-golang-programming-22d451585e07

https://github.com/abdennour/ansible-role-golang

https://stackoverflow.com/questions/35988567/ansible-doesnt-load-profile

https://docs.ansible.com/ansible/latest/user_guide/playbooks_environment.html#playbooks-environment

https://stackoverflow.com/questions/45815938/unable-to-download-golang-repository-by-using-ansible

https://medium.com/learn-go/go-path-explained-cab31a0d90b9

https://docs.ansible.com/ansible/latest/collections/ansible/builtin/shell_module.html#examples

https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html

https://stackoverflow.com/questions/35654286/how-to-check-if-a-file-exists-in-ansible

https://stackoverflow.com/questions/54944080/installing-multiple-packages-in-ansible

https://stackoverflow.com/questions/29289472/ansible-how-ansible-env-path-is-set-in-ssh-session

https://docs.ansible.com/ansible/latest/collections/ansible/builtin/replace_module.html

https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html#return-values

http://api.zeromq.org/4-1:zmq-ctx-term

https://godoc.org/github.com/pebbe/zmq4#Socket.SendBytes

https://stackoverflow.com/questions/28347717/how-to-create-an-empty-file-with-ansible

https://www.guru99.com/file-permissions.html

https://linuxhint.com/vim_split_screen/