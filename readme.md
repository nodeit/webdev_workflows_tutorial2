# Developing reproducible workflow environments with Virtual Machines - Part 2

If you haven't already, start with [Part 1](https://github.com/nodeit/webdev_workflows_tutorial1).

## A better way to provision

In the first tutorial, we setup and provisioned our virtual machine using bash scripts. Although there is nothing wrong with using bash scripts (and I would even recommend them for smaller projects) there is a more modern way to automate your server configuration and application deployment. This is where [Ansible](http://www.ansible.com/get-started) comes in.

Ansible is one of many configuration management and orchestration tools you can choose from to provision your Vagrant machine (and servers). Other popular options are [Chef](https://www.chef.io/chef/), [Puppet](https://puppetlabs.com/), and [Salt](http://saltstack.com/). We will be using Ansible for this tutorial but feel free to check out the other options to see if they suit your situation better.

Although all of these tools are great, I chose to go with Ansible because the configuration is straight-forward and you don't have to install any software on the target machine. As long as you can SSH into the machine, you can provision it with Ansible. Also, Vagrant just released version 1.8 which allows ansible to be run on the target machine instead of the host for windows machines. This is important because Ansible doesn't support using Windows as the host machine. In theory at least, this means this tutorial should work across Linux, Unix and Windows machines. 

The basic idea here is that we will be writing configuration in [YAML](https://en.wikipedia.org/wiki/YAML), which ansible calls "playbooks", instead of raw commands in a shell script. This way, our provisioning configurations will read closer to english and will be self-documenting.

A more powerful feature of Ansible is that, unlike the shell scripts we wrote, it aims to be [idempotent](http://docs.ansible.com/ansible/glossary.html#idempotency) and won't mindlessly run commands against your server without first considering it's current state. 

Ansible's documentations describes the concept of idempotency:

> The concept that change commands should only be applied when they need to be applied, and that it is better to describe the desired state of a system than the process of how to get to that state. As an analogy, the path from North Carolina in the United States to California involves driving a very long way West, but if I were instead in Anchorage, Alaska, driving a long way west is no longer the right way to get to California. Ansible’s Resources like you to say “put me in California” and then decide how to get there. If you were already in California, nothing needs to happen, and it will let you know it didn’t need to change anything.

# Swapping out our bash scripts for Ansible

## Updating our Vagrantfile

We will be running the exact same simple app and static files we did in the first tutorial but this time, we will tell Vagrant to use Ansible as our provisioner. 

To do that, let's remove any references to shell scripts in our Vagrantfile and add the Ansible configuration:

~~~
# Vagrantfile

Vagrant.configure(2) do |config|

  config.vm.box = "ubuntu/trusty64"
  config.vm.network "private_network", ip: "192.168.33.10"
  config.vm.synced_folder "myapp", "/var/www/myapp", type: "rsync"

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "provisioning/playbook.yml"    
  end

end
~~~

Basically, we just told Vagrant that we want to use the "ansible" provider and that the [playbook](http://docs.ansible.com/ansible/playbooks.html) we want to use is located at `~/Projects/tutorial2/provisioning/playbook.yml`. Playbooks let you tell Ansible which machine you want to provision, what user you should connect as, which tasks to run and more. 

Reading over the Ansible documentation can get overwhelming pretty quick because it has a ***lot*** of options. However, keep in mind that you only need to use the features relavent to your project and you can learn new ones as you go. We will be going through all the basic things you need and you can expand your knowledge from there. Ansible has great documentation too so you'll find 99% of what you need there.

## Creating our first playbook

Let's put some basic information in our `playbook.yml` within the provisioning directory of our application:

~~~
---
- hosts: all
  user: vagrant

  roles:
    - nodejs
~~~

If you've never worked with yaml before, this file might look a bit strange. It's somewhat similar in concept to JSON in that you are dealing with key value pairs, but instead of using curly braces, we use indentation to denote structure. Head on over to the [getting started](http://www.yaml.org/start.html) page of yaml to get a better idea. You will be able to pick this syntax up pretty quickly though, especially for our use case.

In our playbook, we are setting three basic things: the host, user and the roles. The host is the machine or machines that you want to run the playbook against. "all" simply means we want to run it against all of the hosts that are in our inventory file. What is this inventory file you ask? Well there is one small (and awesome) caveat when working with Vagrant and Ansible together and that is that Vagrant automatically provides one for you which you can find at `~/projects/tutorial2/.vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory` which will look something like this:

### Inventory file

~~~
# Generated by Vagrant

default ansible_ssh_host=127.0.0.1 ansible_ssh_port=2222 ansible_ssh_private_key_file=/Users/someuser/Projects/tutorial2/.vagrant/machines/default/virtualbox/private_key
~~~

Basically, the inventory file is a list of machines you want to connect to. Eventually, we will update our inventory file so that we can connect to a remote server by adding its IP address and port. If you are interested in reading more on that topic now, head on over to the [inventory documentation](http://docs.ansible.com/ansible/intro_inventory.html).

### Playbook user

The user property we set in the playbook simply tells vagrant that we want to run these commands as the user `vagrant`. We could of course add another user to our vagrant machine and use that, but vagrant makes it convenient for us by automatically providing us with the default vagrant user.

### Playbook roles

One thing you will learn very quickly about Ansible is that everything is organized very neatly in directories. We will be creating a `roles` directory that will hold even more directories and yaml files. This may seem like a lot of work, but once you see how organized it is, it will make sense in the end.

Our roles directory will follow a simple convention:


~~~				            
provisioning
└── roles
    ├── nginx
    │   └── tasks
    │       └── main.yml
    └── nodejs
        └── tasks
            └── main.yml
~~~

**Note:** This is just a small portion of the overall directory structure. Here is the best practices documentation on [directory layout](http://docs.ansible.com/ansible/playbooks_best_practices.html#directory-layout) in ansible. We'll be going over a lot of the basics in this tutorial so don't fret too much.
				
This way, when we write the following in our `playbook.yml`:

~~~
  roles:
    - nodejs
~~~

We are telling Ansible to search in our "nodejs" folder in our roles directory and execute any tasks it finds in the tasks directory.

### Our first task

Create a `main.yml` file and place it in the `~/Projects/tutorial2/provisioning/nodejs/tasks` directory. *You will need to create this directory structure manually*

`main.yml`
~~~
---
- name: Download node.js binary tarball
  get_url: url=https://nodejs.org/dist/v4.0.0/node-v4.0.0-linux-x64.tar.gz dest=/home/vagrant force=yes
  tags: nodejs
~~~

Before I explain this file in more detail, take a moment to notice something; even if this is the first time you've laid eyes on ansible or even a yaml file, it should be pretty clear what we want to happen here. *We want to download the node.js binary file from the web and put it in our `/home/vagrant` directory*

Although this certainly isn't plain english, it's about as close as we're going to get when writing server provising scripts. As you will see, Ansible is self-documenting. 

#### Task name
As you write each "task", you will give it a `name`. This can be anything you'd like but it should be a concise description of what the task should do. These names will appear in your console as the ansible scripts run so you know which tasks are running. These are purely for you

#### Modules
The next thing to note about Ansible is that although you *can* run raw shell scripts, you should instead use the built-in [modules](http://docs.ansible.com/ansible/modules.html). Modules are basically "task plugins" or "library plugins" and they are responsible for doing the actual work. `get_url` is one such module and is a direct replacement for something like `wget` that we used in our bash scripts. 

If you take a look at the [get_url documentation](http://docs.ansible.com/ansible/get_url_module.html), you'll see that we are also using the `force` parameter which states the following:

> If yes and dest is not a directory, will download the file every time and replace the file if the contents change. If no, the file will only be downloaded if the destination does not exist. Generally should be yes only for small local files. Prior to 0.6, this module behaved as if yes was the default.

Since the node.js binary is pretty small at ~11MB, we will set the force parameter to "yes" so that ansible will download the file every time we provision our VM but it will *only replace the file if the contents change*.)

#### Tags

You might be wondering what the "tags" parameter is for. That is simply a way to *group* your tasks so that you can run them specifically instead of the entire playbook. For example, we will have tasks that configure nodejs and nginx; If we want to change something in the nginx tasks, we can tell ansible to only run tasks with the "nginx" tags which can save us some time if you have a lot of tasks.

### Idempotency revisited

As we briefly discussed before, modules strive for idempotency, meaning it only wants to run a task *if something has changed or needs to be updated*. For example, there is no reason to install node *if it is already installed*. In this particular case, that probably wouldn't be a huge deal but some software can't easily be reinstalled or changed without causing problems.

These modules will basically check the current state of your machine before running any commands. Let's see this in action by navigating to our `~/projects/tutorial2` directory and running `vagrant provision` (If your vagrant machine is currently halted, you can run `vagrant up --provision` instead)

~~~
==> default: Checking for host entries
==> default: Running provisioner: ansible...
    default: Running ansible-playbook...

PLAY [all] ********************************************************************

GATHERING FACTS ***************************************************************
ok: [default]

TASK: [nodejs | Download node.js binary tarball] ******************************
changed: [default]

PLAY RECAP ********************************************************************
default                    : ok=2    changed=1    unreachable=0    failed=0
~~~

If we immediately run `vagrant provision` again, we can see vagrant's idempotency in action. Notice how this time, changed will be equal to "0" instead of "1". Basically, ansible saw that we already had this file downloaded and that nothing has changed so it is letting us know that the state of our machine has remained unchanged since the first provision.

~~~
==> default: Checking for host entries
==> default: Running provisioner: ansible...
    default: Running ansible-playbook...

PLAY [all] ********************************************************************

GATHERING FACTS ***************************************************************
ok: [default]

TASK: [nodejs | Download node.js binary tarball] ******************************
ok: [default]

PLAY RECAP ********************************************************************
default                    : ok=2    changed=0    unreachable=0    failed=0
~~~

Now, for illustration purposes only, let's manually ssh into our vagrant machine and delete the binary file that ansible has downloaded for us. This excercise will accomplish a couple of things; it will prove to you that ansible did indeed download the file to the appropriate directory and it will change the state of the machine so that we can see how ansible will react to that change. 

SSH into our VM and navigate to our home directory and find the file:

~~~
vagrant ssh
cd ~
ls -lh
~~~

You should see the tarball in that directory, let's go ahead and remove that and then logout of the VM

~~~
sudo rm node-v4.2.4-linux-x64.tar.gz
ls # file should be empty
logout
~~~

Now that we are back on the host machine, run `vagrant provision` again and see what happens:

~~~
==> default: Checking for host entries
==> default: Running provisioner: ansible...
    default: Running ansible-playbook...

PLAY [all] ********************************************************************

GATHERING FACTS ***************************************************************
ok: [default]

TASK: [nodejs | Download node.js binary tarball] ******************************
changed: [default]

PLAY RECAP ********************************************************************
default                    : ok=2    changed=1    unreachable=0    failed=0
~~~

You'll notice that ansible detected the change and let you know about it. Just to be clear, you don't want to go into the VM and manually install/remove environment settings. This was purely for demonstration. The only things you'll want to run/install manually on the machine are app related things such as running the app or installing it's depencies (which are in your package.json file).

#### Fully installing node.js

Now that we've got the basic idea down, let's go ahead and fully install node.js. We'll introduce some new helpful ansible modules and techniques while we are at it. 

Now that we have some configuration that installs node, the next step is to unarchive the tarball file and move it to our /usr/local directory. 

`main.yml`
~~~
---
- name: Download node.js binary tarball
  get_url: url=https://nodejs.org/dist/v4.0.0/node-v4.0.0-linux-x64.tar.gz dest=/home/vagrant force=yes
  tags: nodejs

- name: Unarchive node.js tarball
  unarchive: src=/home/vagrant/node-v4.0.0-linux-x64.tar.gz dest=/usr/local copy=no
  tags: nodejs
~~~

Here, we are using the [unarchive](http://docs.ansible.com/ansible/unarchive_module.html) module in place of `tar` in our old shell scripts along with the `copy=no` option. If copy is set to "yes", it will attempt to copy a file from your *host* machine instead of the VM, which we obviously don't want.

Next, we will symlink our node binary to the /usr/local/node directory. we can accomplish this using the [file](http://docs.ansible.com/ansible/file_module.html) module with `state` option set to "link".

`main.yml`
~~~
---
- name: Download node.js binary tarball
  get_url: url=https://nodejs.org/dist/v4.0.0/node-v4.0.0-linux-x64.tar.gz dest=/home/vagrant force=yes
  tags: nodejs

- name: Unarchive node.js tarball
  unarchive: src=/home/vagrant/node-v4.0.0-linux-x64.tar.gz dest=/usr/local copy=no
  tags: nodejs
  
- name: Add /usr/local symlink to unarchived tarball
  file: src=/usr/local/node-v4.0.0-linux-x64 dest=/usr/local/node state=link
  tags: nodejs
~~~

And finally, we will want to update our `.profile` so that it can add /usr/local/node to our path. 

`main.yml`
~~~
---
- name: Download node.js binary tarball
  get_url: url=https://nodejs.org/dist/v4.0.0/node-v4.0.0-linux-x64.tar.gz dest=/home/vagrant force=yes
  tags: nodejs

- name: Unarchive node.js tarball
  unarchive: src=/home/vagrant/node-v4.0.0-linux-x64.tar.gz dest=/usr/local copy=no
  tags: nodejs
  
- name: Add /usr/local symlink to unarchived tarball
  file: src=/usr/local/node-v4.0.0-linux-x64 dest=/usr/local/node state=link
  tags: nodejs
  
- name: Add ~/.profile with updated node path
  template: src=profile dest=/home/vagrant/.profile
  tags: nodejs
~~~

Remember in our shell scripts where we manually updated our profile like so:

~~~
# add node to path
echo 'export PATH=/usr/local/node/bin:$PATH' >> ~/.profile
~~~

While there is nothing wrong with this, Ansible gives us a cleaner approach (in my opinion) to acomplish this through [templates](http://docs.ansible.com/ansible/template_module.html).

So instead of writing our `.profile` updates directly in our configuration code, we can write a template that is neatly organized in its own directory. 

Create a new file called `profile` and place it in our `~/Projects/tutorial2/provisioning/roles/nodejs/templates` directory:

**NOTE**: We are purposely leaving off the `.` in the profile so that our host file-system doesn't hide it from us. Also, we have instructed Ansible to add the dot later with this `dest=/home/vagrant/.profile`

`profile`

~~~
# Add node to path
PATH="/usr/local/node/bin:$PATH"

# if running bash
if [ -n "$BASH_VERSION" ]; then
    # include .bashrc if it exists
    if [ -f "$HOME/.bashrc" ]; then
    . "$HOME/.bashrc"
    fi
fi

# set PATH so it includes user's private bin if it exists
if [ -d "$HOME/bin" ] ; then
    PATH="$HOME/bin:$PATH"
fi
~~~

What we are doing here is telling ansible how we want our *entire* `.profile` to look instead of just updating a part of it. So getting back to our tasks, the following line `template: src=profile dest=/home/vagrant/.profile` is telling Ansible to take our profile template we just created and replace the default profile. 

**Note:** If you are wondering where the rest of the code in our profile came from, I simply copied the default profile in the VM and then added the node path.

#### Provision and check node installation

Now we are ready to provision our VM again and completely install node. Do that by running the following on your host machine in the directory where your Vagrantfile is: 

~~~
vagrant up
~~~

You should see the following:

~~~
==> default: Checking for host entries
==> default: Running provisioner: ansible...
    default: Running ansible-playbook...

PLAY [all] ********************************************************************

GATHERING FACTS ***************************************************************
ok: [default]

TASK: [nodejs | Download node.js binary tarball] ******************************
ok: [default]

TASK: [nodejs | Unarchive node.js tarball] ************************************
changed: [default]

TASK: [nodejs | Add /usr/local symlink to unarchived tarball] *****************
changed: [default]

TASK: [nodejs | Add ~/.profile with updated node path] ************************
changed: [default]

PLAY RECAP ********************************************************************
default                    : ok=5    changed=3    unreachable=0    failed=0
~~~


Let's check our installation by SSH'ing into the VM and checking node's version:

`host machine`

~~~
vagrant ssh
~~~

`VM`

~~~
node -v
>>> v4.2.4
~~~

If you get the node version, everything has been installed correctly!

#### Variables

So far, our nodejs playbook is pretty straightforward but it can definitely be improved a bit. For example, what happens when you want to upgrade to the newest node version next week or run this playbook as a different ssh user, say on our production server?

That's where [variables](http://docs.ansible.com/ansible/playbooks_variables.html) come in. There are a few ways to set variables including in the playbook itself, in a variable directory, inventory file or even in our Vagrantfile. Since we are using Vagrant, let's start by declaring some variables in the Vagrantfile:

In our `Vagrantfile`, let's update the provisioning section to look like this:

~~~
  config.vm.provision "ansible" do |ansible|
    
    ansible.sudo = true
    ansible.playbook = "provisioning/playbook.yml"

    ansible.extra_vars = {
      node: {
        version: "4.2.4"
      }
    }

  end
~~~

All we are doing here is setting setting some extra parameters in our Vagrant configuration that will passed into Ansible via Vagrant. These variables will now be available to us in our playbook tasks. If you are familiar with javascript objects, the notation here is virtually the same. You can nest key value pairs in the same way.

Now, let's use this new node version variable to abstract our playbook a bit:

`nodejs/tasks/main.yml`
~~~
---
- name: Download node.js binary tarball
  get_url: url=https://nodejs.org/dist/v{{node.version}}/node-v{{node.version}}-linux-x64.tar.gz dest=/home/vagrant force=yes
  tags: nodejs

- name: Unarchive node.js tarball
  unarchive: src=/home/vagrant/node-v{{node.version}}-linux-x64.tar.gz dest=/usr/local copy=no
  tags: nodejs

- name: Add /usr/local symlink to unarchived tarball
  file: src=/usr/local/node-v{{node.version}}-linux-x64 dest=/usr/local/node state=link
  tags: nodejs

- name: Add ~/.profile with updated node path
  template: src=profile dest=/home/vagrant/.profile
  tags: nodejs
~~~

We are simply replacing all instances of the node version in our playbook with the `{{ node.version }}` variable.

So, if we want to change node versions, we don't have to touch our playbook anymore, we can simply change variable in the Vagrantfile and re-provision our VM. Try it now by changing the version to `4.0.0` and then running `vagrant provision`. 

If all went well you can ssh back into the machine and check the node version again to confirm it changed to `v4.0.0`.

#### Provisioning nginx

To get our ansible config up to speed with our old shell scripts we still need to install nginx. We will be using the exact same techniques as we did with nodejs but I will also introduce a couple more ansible concepts that you can use in your own projects if you wish.

Let's start by creating a directory for our nginx plays and making a new main.yml file for our tasks:

~~~
mkdir -p ~/Projects/tutorial2/provisioning/roles/nginx/tasks
touch ~/Projects/tutorial2/provisioning/roles/nginx/tasks/main.yml
~~~

Place the following in our main.yml:

~~~
---
- name: Install nginx
  apt: pkg=nginx-full
  tags: nginx

- name: Update default sites-available file
  template: src=default dest=/etc/nginx/sites-available/
  tags: nginx
  notify:
    - restart nginx

~~~

By now, it should be pretty self-explanatory what's going on. We are installing nginx, placing a default template in our `sites-available` directory.

You'll also notice that we are using a new parameter here called `notify`. Basically, notify will run what is called a `handler`, which is simply another task, *only if this current task has changed*. For example, if ansible notices that your template for `sites-available` has changed, it will run the handler tasks. Let's make those now:

Create a new directory for your nginx handlers and place a new `main.yml` file within it:

~~~
cd ~/Projects/tutorial2/provisioning/roles/nginx
mkdir handlers
touch handlers/main.yml
~~~

Within that new `main.yml` file, add this:

~~~
---
- name: restart nginx
  service: name=nginx state=restarted
~~~

You'll notice that this is really no different than our previous tasks. We give it a name and use a module ([service](http://docs.ansible.com/ansible/service_module.html) in this case). We are omitting the `tags` parameter because handlers are only called from other tasks which will already have tags. We won't be calling it on it's own anyway. 

This handler task simply restarts the nginx service.

#### Updating our playbook.yml

Now that we have our nginx tasks in place, we need to update our `playbook.yml` file and tell it to run the nginx tasks:

Update `~/Projects/tutorial2/provisioning/playbook.yml`:

~~~
---
- hosts: all
  user: vagrant

  roles:
    - nginx
    - nodejs
~~~

Now re-provision the VM with:

~~~
>>> vagrant provision

==> default: Checking for host entries
==> default: Running provisioner: ansible...
    default: Running ansible-playbook...

PLAY [all] ********************************************************************

GATHERING FACTS ***************************************************************
ok: [default]

TASK: [nginx | Install nginx] *************************************************
changed: [default]

TASK: [nginx | Update default sites-available file] ***************************
changed: [default]

TASK: [nodejs | Download node.js binary tarball] ******************************
ok: [default]

TASK: [nodejs | Unarchive node.js tarball] ************************************
ok: [default]

TASK: [nodejs | Add /usr/local symlink to unarchived tarball] *****************
ok: [default]

TASK: [nodejs | Add ~/.profile with updated node path] ************************
ok: [default]

NOTIFIED: [nginx | restart nginx] *********************************************
changed: [default]

PLAY RECAP ********************************************************************
default                    : ok=9    changed=4    unreachable=0    failed=0
~~~

















