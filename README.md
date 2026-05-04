# Ansible role and playbook for Heriverse

This repository includes an [Ansible](https://ansible.com) role and sample playbook to install an instance of the Heriverse web application in a production environment.

Since Ansible is a configuration management and provisioning system, the role provided here can be used to deploy the application on a number of machines simultaneously, with some caveats described below.

Ansible uses standard YAML syntax for configuration files and the [Jinja 2](https://jinja.palletsprojects.com/en/stable/) syntax for templates.

## Assumptions

At present, the Ansible role assumes that Docker is available on the target host, so it doesn't attempt to install it, although this could be added (see [TODO](#todo)).

The particular operating system / Linux distribution installed on the target host shouldn't make any difference, although further testing is required.

## Role

The role adopts the standard folder structure recognized by Ansible, which is as follows:

```
role
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
└── templates
    ├── Caddyfile.j2
    └── docker-compose.yml.j2
```

The `main.yml` file in the `tasks` folder includes the sequence of tasks that Ansible should perform when the playbook is executed. These tasks reference the variable values defined in `defaults/main.yml` and the template files in `templates`.

### Default variables

The variables values defined in `defaults/main.yml` are related to the URL of the Git repo for the Heriverse Docker Compose installation, maintained by 3DResearch, and to the configurable server name where the application should be available.

### Templates

The Jinja templates included here are intended as replacements, with Ansible variables, for the Heriverse `docker-compose.yml` file and Caddy's configuration (`Caddyfile`).

The reason for including these is mostly due to the need for choosing an arbitrary server name (domain) for the Heriverse installation.

When the relevant task is executed, Ansible overwrites these two file with the interpolated templates.

## Sample playbook

The playbook provided here is a very simple example of how the Heriverse role can be used for deployment in a production environment. It makes basic assumptions about the availability of the role in standard Ansible paths and shows how the default variables discussed above can be overridden.

It can be launched with, e.g.,

```shell
ansible-playbook heriverse-ansible/playbook/heriverse.yml
```

## TODO

- [ ] Describe tasks in more detail
- [ ] Include a Docker installation role as well?
- [ ] Avoid replacing the original `docker-compose.yml` with the template, if possible.
