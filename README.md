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

## What this deploys, beyond Heriverse

The templates here bring up the **whole StratiGraph system** behind one Caddy and
one certificate — nine containers, routed by path:

| route | container | what it is |
|---|---|---|
| `/`, `/server/*` | heriverse, heriverse-server | the 3D front end and its API |
| `/auth/*` | keycloak | one realm for everything |
| `/couchdb/*` | couchdb | documents (scenes, and the Catalog's index) |
| `/em/*` | **em-server** | the ROOM: the live graph, its assets, the WebSocket |
| `/catalog/*` | **em-catalog** | the REGISTER: studies as published |
| `/iiif/*` | cantaloupe | the IIIF Image API, straight out of the bucket |
| `/assets/*` | minio | the object store |

Each StratiGraph service is behind a flag (`em_server_enabled`,
`em_catalog_enabled`, `minio_enabled`, `iiif_enabled`,
`em_catalog_couchdb_enabled`), so a deployment that wants only Heriverse turns
them off and the templates render without them.

**No credentials live in this repository.** MinIO's and CouchDB's come from the
inventory, a Vault file or the environment; if they are missing the template
**fails loudly** rather than falling back to a default password. That failure is
the feature.

### The reading page (`em_catalog_reader_dist`)

A study can be read as a story at `/catalog/study/{id}/narrative`. The page that
renders it is EMStudio's **reader**, and em-catalog serves it under
`/catalog/reader/` — which falls inside the `/catalog/*` route above, so there is
nothing to add to the Caddyfile.

It is a **directory**, not a file: the reader gave up its single-file build so
its 3D engine could be a chunk fetched only when a model appears. The shell and
its `assets/` must be served from the same place, and its asset paths are
relative, so the same build works under any prefix without being rebuilt for a
particular host.

```yaml
em_catalog_reader_dist: "/opt/heriverse/reader-dist"   # a path ON THE HOST
```

Set → the compose gains `EM_CATALOG_READER` and a read-only bind mount, and the
play **refuses to continue** if that path has no `reader.html` (a variable
pointing at nothing would otherwise let Docker create an empty directory and
serve a 501 while the configuration claims a reader).

Empty — **the default** — → no env, no mount, and the narrative route answers an
honest **501** naming the build command. Not a blank page: a blank page reads as
an empty study.

How the directory gets there is deliberately **not** decided here. Today:

```shell
cd EMStudio/frontend && npm run build:reader     # → dist/ : reader.html + assets/
rsync -a dist/ <host>:/opt/heriverse/reader-dist/
```

Replacing it later is that `rsync` and nothing else — em-catalog stats the files
per request, so a new build is served without a restart. Baking the reader into
the em-catalog image, or building it on the host, would tie the Catalog's release
to the front end's; publishing the artefact properly is WP6's pipeline to build
(CNR writes the spec, 3DR builds). This role's job is to be *capable* of serving
it.

* **How the pieces fit together:** [`ARCHITECTURE-SYSTEM.md`](../em-server/docs/ARCHITECTURE-SYSTEM.md)
* **Prerequisites, the command, and what to check afterwards:** [`DEPLOYMENT.md`](../em-server/docs/DEPLOYMENT.md)
* **Which URL is internal and which is public:** [`URL-TOPOLOGY.md`](../em-server/docs/URL-TOPOLOGY.md)

### Checking a change without a host

Both templates can be judged by the tools that will consume them, with no
deployment. Verified this way:

```shell
# render docker-compose.yml.j2 with example values, then:
docker compose -f /tmp/rendered.yml config          # → VALID, 9 services
# render Caddyfile.j2, then:
docker run --rm -v "$PWD:/work:ro" caddy:2-alpine \
  caddy validate --config /work/Caddyfile --adapter caddyfile   # → Valid configuration
ansible-playbook --syntax-check playbook/heriverse.yml
```

Two notes for whoever runs the last one: this repo has `role/` (singular), so the
playbook cannot resolve the role named `heriverse` without an
`ANSIBLE_ROLES_PATH` pointing at a directory that contains it under that name (or
an `ansible.cfg`); and the role uses `community.docker.docker_compose_v2`, so
that collection has to be installed.

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
