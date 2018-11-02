## Introduction

This site is meant to provide a tutorial that is an as-simple-as-possible introduction to Ansible, for those unfamiliar with it or new to it.

The instructions below therefore follow a very simple, prescriptive approach to using Ansible, without covering the wealth of options and configurability Ansible allows. For those interested in these wider options, feel free to follow the provided links to read the much more in-depth Ansible guides for those concepts.

Why Ansible?

1. It's free.
1. Does not require _anything_ to be installed on servers -- it's just SSH and Python.
1. We can define tasks in a high-level language (with 1000+ pre-built operations).
1. It's operations are [idempotent](https://en.wikipedia.org/wiki/Idempotence).
1. It can be used to deploy across platforms (Linux and Windows).

And perhaps most importantly, the "code" (3) is [YAML](http://yaml.org) -- which is simple, structured text documentation _that's also executable_.

## Basic concepts

### Inventory

An Ansible [**inventory**](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html) is a simple text file in which details about the various servers to which you will deploy are defined.

The [**inventory**](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html) file can actually be defined in various formats, including `ini` and `yaml`, and can even be [dynamically-constructed](https://docs.ansible.com/ansible/latest/user_guide/intro_dynamic_inventory.html) eg. from a cloud service.

Perhaps the simplest (and most common) form of an inventory is `ini`-style:

```ini
[group]
hostname
```

Each entry in an `ini`-style inventory is headed by a `group`, in square-brackets, under which all of the servers that should be part of that group are listed (by hostname). The `group` names themselves are arbitrary, and specific only to the playbooks you are deploying. A complex playbook -- one that deploys across potentially many different servers -- may have many groups.

For example, a simple inventory for a classical 3-tier web application might be:

```ini
[balancer]
load.somewhere.com

[appserver]
app.somewhere.com

[database]
db.somewhere.com
```

The inventory is also used to define any connectivity details for a given connection.

```ini
[balancer]
load.somewhere.com ansible_user='abc' ansible_ssh_private_key_file='~/.ssh/id_abc'
```

> The example above defines that load.somewhere.com should be connected to using SSH, as user `abc`, and using a private key in `~/.ssh/id_abc`.

```ini
[appserver]
app.somewhere.com ansible_user='administrator' ansible_password='MyPassw0rd' ansible_port=5986 ansible_connection=winrm ansible_winrm_server_cert_validation=ignore
```

> The example above defines that app.somewhere.com should be connected to using WinRM (a Windows host), as user `administrator`, using password `MyPassw0rd`, through port `5986`, and that it should ignore SSL certificate validation (ie. accept a self-signed certificate from the host).

The [Ansible documentation](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#list-of-behavioral-inventory-parameters) provides a full set of variables that can be used to configure connectivity options.

### Roles

Before we get to playbooks, it is worth understanding one of the basic modularisation mechanisms that Ansible supports: the [**role**](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html).

Roles provide a mechanism through which all of the tasks needed to deploy a particular "thing" can be combined together in a re-usable way for others to make use of, simply by referring to the role within their playbook. As such, they abstract away the need to know all of the underlying task-by-task details of deployment of that "thing".

For example, you may have one role that defines how to deploy a particular piece of database software like [`bernardovale.db2`](https://galaxy.ansible.com/bernardovale/db2) and another that defines how to deploy a load balancer [`nginxinc.nginx`](https://galaxy.ansible.com/nginxinc/nginx).

While roles themselves are implemented through `YAML`, and therefore are typically easy to read and understand, they are generally intended to be used as-is by a user without modification. Any inputs that may vary in your particular environment will typically be catered for through variables.

Ansible provides a package-manager-of-sorts for roles called Galaxy, so a role can be easily installed using a command like:

```shell
$ ansible-galaxy install nginxinc.nginx
```

### Playbook

An Ansible [**playbook**](https://docs.ansible.com/ansible/latest/user_guide/playbooks.html) contains the instructions for deploying something.

Playbooks typically:

- make use of one or more roles, for modularity.
- rely on an inventory to define where the portions of what is deploying will be deployed (ie. on what servers).
- make use of one or more variables to parameterise how they deploy or configure their "something".

Playbooks are written in `YAML`, and generally follow the structure:

```yml
---
- name: Deploy DB2 to database hosts
  hosts:
    - database
  roles:
    - bernardovale.db2

- name: Deploy NGINX to load-balancer hosts
  hosts:
    - balancer
  roles:
    - nginxinc.nginx
```

This example playbook would first ensure that DB2 is deployed to all hosts under the `[database]` group of the inventory, and then ensure that NGINX is deployed to all hosts under the `[balancer]` group of the inventory. The actual set of tasks needed for each of these deployments is simply pulled-in from the [`bernardovale.db2`](https://galaxy.ansible.com/bernardovale/db2) role and the [`nginxinc.nginx`](https://galaxy.ansible.com/nginxinc/nginx) role, respectively.

Playbooks can be run through the `ansible-playbook` command, passing the inventory and playbook name:

```shell
$ ansible-playbook -i inventory_filename playbook.yml
```

As the playbook runs, it will print out information about what task it is carrying out, against which server. Green "ok" indicates that the task has completed without making any change, yellow "changed" indicates that the task has completed and made some change to the system, and red "failed" indicates that there was an error. (By default, any host that has a "failed" task will no longer have any tasks run against it for the remainder of the playbook.)

### Variables

As mentioned above, [**variables**](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html) are generally used by both roles and playbooks to provide parameterisation to the uniqueness of your particular environment.

Common examples include things like directory locations and user credentials.

It is considered good practice for roles to document the minimal set of variables they expect, as well as any further (optional) ones that may influence how the role deploys its "thing".

There are many ways to [pass variables through to Ansible](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html), and if you decide to opt for using many approaches at the same time it is important to understand their precedence. If you are creating your own variables, make sure you also avoid the use of [reserved words](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#what-makes-a-valid-variable-name).

However, for simplicity, often the easiest way to make use of variables in a consistent way is through one (or both) of the following two methods. These will then be automatically picked up each time when running the playbook.

#### `group_vars`

Creating a sub-directory (relative to your playbook) called `group_vars`, in which you place a single file called `all.yml`. This single file can then contain all of the variables you want to configure, across all of the groups involved in your playbook.

```yml
---
db2_binary:
    url: https://mycompany.com/downloads/db2_10_5.tar.gz
    location: /ansible/files/db2_10.5.tar.gz
    dest: /tmp

nginx_type: opensource
nginx_install_from: nginx_repository
```

#### `host_vars`

If you want to override a particular variable for a single host, often this is most easily done through a `host_vars` sub-directory (relative to your playbook), in which you place a file for each server for which you want to override variables. (The filename should be `<hostname>.yml`.)

```yml
nginx_unit_enable: true
```

## Securing sensitive information

A common concern when deploying technology is ensuring that sensitive information (like user credentials) are appropriately protected.

Ansible addresses this concern through its [**vault**](https://docs.ansible.com/ansible/latest/user_guide/vault.html) capability. The vault allows you to encrypt the contents of a file defining such sensitive variables, so that using them requires a password to decrypt them.

A common pattern to make use of a vault is to replace any sensitive information in your simple variable files (eg. `group_vars/all.yml`) with other variables. For example:

```yml
---
db2_instance_owner: "{{ vault_db2_instance_owner }}"
db2_instance_owner_pwd: "{{ vault_db2_instance_owner_pwd }}"
```

Create a separate file that contains only the sensitive information (eg. `group_vars/vault.yml`):

```yml
---
vault_db2_instance_owner: db2inst1
vault_db2_instance_owner_pwd: myDb2Inst1Passw0rd
```

And encrypt this separate file using Ansible vault:

```shell
$ ansible-vault encrypt group_vars/vault.yml
```

Ensure that your playbook includes this vault as well, by simply adding a `vars_file` line:

```yml
---
- name: Deploy DB2 to database hosts
  hosts:
    - database
  roles:
    - bernardovale.db2
  vars_file:
    - vault.yml
```

And then when you run your playbook, simply ensure it asks for the vault password:

```shell
$ ansible-playbook -i inventory_filename playbook.yml --ask-vault-pass
```
