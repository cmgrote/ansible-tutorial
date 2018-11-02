# Introduction

This site provides a simple introduction to Ansible, for those unfamiliar with or new to it.

The instructions below are intentionally as simple and prescriptive as possible, without covering the wealth of options and configurability Ansible allows. For those interested in further options, follow the provided links to the in-depth Ansible guides for those concepts.

Why Ansible?

1. It's free.
1. Does not require _anything_ to be installed on servers -- it's just SSH and Python.
1. We can define tasks in a high-level language (with 1000+ pre-built operations).
1. Its operations are [idempotent](https://en.wikipedia.org/wiki/Idempotence).
1. It can deploy across platforms (Linux and Windows).

And perhaps most importantly, the "code" (3) is [YAML](http://yaml.org) -- simple, structured text documentation _that's also executable_.

# Basic concepts

## Inventory

An Ansible [**inventory**](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html) is a simple text file that details the various servers onto which to deploy.

The [**inventory**](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html) file can be defined in various formats and can even be [dynamically-constructed](https://docs.ansible.com/ansible/latest/user_guide/intro_dynamic_inventory.html), eg. from a cloud service.

Perhaps the simplest (and most common) form of an inventory is ini-style:

```ini
[group]
hostname
```

- Each entry in an ini-style inventory is headed by a `group`, in square-brackets.
- Under each group is a list of the servers that should be part of that group (by hostname).
- The `group` names themselves are arbitrary, and specific only to the playbooks you are deploying.
- A complex playbook -- one that deploys across potentially many different servers -- may have many groups.

For example:

```ini
[balancer]
load.somewhere.com

[appserver]
app.somewhere.com

[database]
db.somewhere.com
```

> This inventory defines three groups: balancer, appserver, and database. Each group has a single, distinct host.

The inventory also defines any connectivity details for a given host:

```ini
[balancer]
load.somewhere.com ansible_user='abc' ansible_ssh_private_key_file='~/.ssh/id_abc'
```

> This inventory connects to load.somewhere.com using SSH, as user `abc`, and using a private key in `~/.ssh/id_abc`.

```ini
[appserver]
app.somewhere.com ansible_user='administrator' ansible_password='MyPassw0rd' ansible_port=5986 ansible_connection=winrm ansible_winrm_server_cert_validation=ignore
```

> This inventory connects to app.somewhere.com using WinRM (Windows Remote Management), as user `administrator`, using password `MyPassw0rd`, through port `5986`, and ignores SSL certificate validation (ie. accepts a self-signed certificate from the host).

The [Ansible documentation](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#list-of-behavioral-inventory-parameters) provides a full set of variables that can be used to configure connectivity.

## Roles

Before we get to playbooks, it is worth understanding one of the basic modularisation mechanisms of Ansible: the [**role**](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html).

Roles provide a way to package together all of the tasks needed to deploy a particular "thing" in a re-usable way, for others to make use of simply by referring to the role within their playbook. As such, roles abstract away the need to know all of the underlying task-by-task details of deployment of that "thing".

For example, you may have one role that deploys a particular piece of database software like [`bernardovale.db2`](https://galaxy.ansible.com/bernardovale/db2) and another that deploys a load balancer [`nginxinc.nginx`](https://galaxy.ansible.com/nginxinc/nginx).

While roles themselves are implemented through `YAML`, and therefore are typically easy to read and understand, they are generally intended to be used as-is (without modification). Any inputs that may vary in your particular environment will typically be catered for through variables.

Ansible provides a package-manager-of-sorts for obtaining roles called Galaxy, so a role can be easily installed using a command like:

```shell
$ ansible-galaxy install nginxinc.nginx
```

## Playbook

An Ansible [**playbook**](https://docs.ansible.com/ansible/latest/user_guide/playbooks.html) contains the instructions for deploying something.

Playbooks typically:

- Make use of one or more roles, for modularity.
- Rely on an inventory to know where (ie. on what servers) to deploy different portions of the "something" being deployed.
- Make use of one or more variables to parameterise how they deploy or configure their "something".

Playbooks are written in `YAML`, and generally follow a simple structure defining these points:

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

> This example playbook would first ensure that DB2 is deployed to all hosts under the `[database]` group of the inventory, and then ensure that NGINX is deployed to all hosts under the `[balancer]` group of the inventory. The actual set of tasks needed for each of these deployments is simply pulled-in from the [`bernardovale.db2`](https://galaxy.ansible.com/bernardovale/db2) role and the [`nginxinc.nginx`](https://galaxy.ansible.com/nginxinc/nginx) role, respectively.

Playbooks can be run through the `ansible-playbook` command, passing the inventory and playbook name:

```shell
$ ansible-playbook -i inventory_filename playbook.yml
```

As the playbook runs, it will print out information about what task it is carrying out, against which server. A green "ok" indicates that the task has completed without making any change, a yellow "changed" indicates that the task has completed and made some change to the system, and a red "failed" indicates that there was an error. (By default, any host that has a "failed" task will no longer have any tasks run against it for the remainder of the playbook.)

A summary, by host, of the count of "ok", "changed", and "failed" tasks will also be printed at the end of the playbook's run.

## Variables

As mentioned above, [**variables**](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html) are generally used by both roles and playbooks to provide parameterisation to the uniqueness of your particular environment.

Common examples include things like directory locations, enabling or disabling optional features, and user credentials.

It is considered good practice for roles to document the minimal set of variables they expect, as well as any further (optional) ones that may influence how the role deploys its "thing". When this is not done, typically the variables can be found inside a role by looking at that roles' `defaults/main.yml` file.

There are many ways to [pass variables through to Ansible](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html), and if you decide to opt for using many approaches at the same time it is important to understand their precedence. If you are creating your own variables, make sure you also avoid the use of [reserved words](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#what-makes-a-valid-variable-name).

However, for simplicity, often the easiest way to make use of variables in a consistent way is through one (or both) of the following two methods. These will then be automatically picked up each time when running the playbook.

### group_vars

Create a sub-directory (relative to your playbook) called `group_vars`, in which you place a single file called `all.yml`. This single file can then contain all of the variables you want to configure, across all of the groups involved in your playbook.

```yml
---
db2_binary:
  url: https://mycompany.com/downloads/db2_10_5.tar.gz
  location: /ansible/files/db2_10.5.tar.gz
  dest: /tmp

nginx_type: opensource
nginx_install_from: nginx_repository
```

> In this example, the location of DB2 binaries and some NGINX options are defined, both taken from the documentation of these actual roles.

### host_vars

If you want to override a particular variable for a single host, often this is most easily done through a `host_vars` sub-directory (relative to your playbook), in which you place a file for each server for which you want to override variables. (The filename should be `<hostname>.yml`.)

```yml
---
nginx_unit_enable: true
```

> In this example, the `nginx_unit_enable` has been switched on for only one particular host (controlled by the name of the file).

# Securing sensitive information

A common concern when deploying is ensuring that sensitive information (like user credentials) is appropriately protected.

Ansible addresses this concern through its [**vault**](https://docs.ansible.com/ansible/latest/user_guide/vault.html) capability. The vault allows you to encrypt the contents of a file defining such sensitive variables, so that using them requires a password to decrypt them.

A common pattern to make use of a vault is to replace any sensitive information in your simple variable files (eg. `group_vars/all.yml`) with other variables. For example:

```yml
---
db2_instance_owner: "{{ vault_db2_instance_owner }}"
db2_instance_owner_pwd: "{{ vault_db2_instance_owner_pwd }}"
```

Then create a separate file that contains only the sensitive information (eg. `group_vars/vault.yml`):

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
