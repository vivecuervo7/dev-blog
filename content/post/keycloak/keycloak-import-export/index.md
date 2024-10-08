---
title: Importing and exporting Keycloak configuration
description: A guide and sample code for importing and exporting Keycloak realms
slug: keycloak-import-export
date: 2024-08-05 00:00:00+0000
image: cover.jpg
categories:
  - Authentication
tags:
  - Keycloak
  - Docker
weight: 2
links:
  - title: Keycloak
    description: Open Source Identity and Access Management
    website: https://www.keycloak.org/
  - title: Docker
    description: Accelerated Container Application Development
    website: https://www.docker.com/
---

## Context

Keycloak allows us to import and export realms, which can make it much easier to share configurations amongst team members.

## Exporting an existing realm

The following instructions to export a realm from Keycloak will assume the use of a docker compose file similar to this.

```yaml
services:
  keycloak:
    image: quay.io/keycloak/keycloak:25.0.2
    container_name: keycloak
    ports:
      - 8080:8080
    environment:
      - KEYCLOAK_ADMIN=admin
      - KEYCLOAK_ADMIN_PASSWORD=admin
    volumes:
      - keycloak-data:/opt/keycloak/data/
    restart: always
    command:
      - "start-dev"
volumes:
  keycloak-data:
    name: keycloak-data
```

The easiest way to export a realm when using docker compose is to add a second compose file. Call this `docker-compose.export.yml`.

```yaml
services:
  keycloak:
    command: "export --dir /opt/keycloak/data/export/ --realm local-dev --users realm_file"
    volumes:
      - ./output:/opt/keycloak/data/export
```

Run the following in your terminal to export the configured realm.

```bash
docker-compose -f "docker-compose.yml" -f "docker-compose.export.yml" up --exit-code-from keycloak
```

You should now have a directory called `output` that contains a file called `local-dev-realm.json`. This file can be imported manually when creating a new realm, or Keycloak can be configured to automatically import this realm when the service starts (attempting to import an already-existing realm will fail to prevent overwrites).

An important caveat to note is that Keycloak is designed to export from a _stopped_ server, meaning you will need to ensure that your configuration has been persisted through some means.

Keycloak also allows for multiple options when it comes to if and how users should be exported. The example above uses the simpler approach of combining them into the realm file. See the [Keycloak documentation](https://www.keycloak.org/server/importExport) for more details.

## Importing a realm from a file

The updated Docker compose file below uses a bind mount to an `import` directory. Once any `realm.json` files have been exported, placing the files in the `import` directory will allow Keycloak to automatically pick those realms up and import them on first run.

Update your `docker-compose.yml` to the following and run by calling `docker-compose up -d` in your terminal.

```yaml
services:
  keycloak:
    image: quay.io/keycloak/keycloak:25.0.2
    container_name: keycloak
    ports:
      - 8080:8080
    environment:
      - KEYCLOAK_ADMIN=admin
      - KEYCLOAK_ADMIN_PASSWORD=admin
    volumes:
      - ./import:/opt/keycloak/data/import
    restart: always
    command:
      - "start-dev"
      - "--import-realm"
```

Note the removal of the persisted Docker volume &mdash; this effectively gives us an auth server that can be modified on the fly, but will reset back to the exported realm whenever it restarts.

As Keycloak won't overwrite an existing realm with the import method, the Docker volume can always be reintroduced and will essentially mean our import file serves as a starting point upon which changes can be persisted.
