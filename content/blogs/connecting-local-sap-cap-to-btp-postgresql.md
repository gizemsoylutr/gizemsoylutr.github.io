---
title: "Connecting a Local SAP CAP Application to SAP BTP PostgreSQL in Hybrid Profile"
date: 2026-01-15T10:00:00+03:00
draft: false
tags: ["SAP", "BTP", "CAP", "PostgreSQL", "Hybrid"]
categories: ["Tutorial"]
author: "Gizem Nur Soylu"
---

## Introduction

This document explains how to connect a local SAP CAP (Cloud Application Programming Model) application to a PostgreSQL database running on SAP BTP (PostgreSQL – Hyperscaler Option).

Before proceeding, the following prerequisites are assumed to be fulfilled:

- A PostgreSQL (Hyperscaler Option) service instance has already been created on SAP BTP.
- A Service Instance Key has been generated for this PostgreSQL instance.
- The developer has access to the instance key credentials in JSON format (including database name, host, port, user, password, and SSL certificate).

The screenshot below illustrates:

- An existing PostgreSQL service instance (`your-posgresql-instance`)
- A corresponding service key (`your-posgresql-instance-key`)
- The credential details exposed via the service key (e.g. `dbname`, `hostname`, `port`, `username`, `password`, `sslcert`)

## Establish an SSH Tunnel to the PostgreSQL Instance

The PostgreSQL (Hyperscaler Option) service on SAP BTP is deployed within the internal Cloud Foundry network and is not publicly accessible. As a result, a local development environment cannot connect to the database directly.

To enable local access, an SSH tunnel must be established via a Cloud Foundry application that has network-level access to the PostgreSQL instance.

This is achieved using the following command:

```bash
cf ssh -L 63306:<your_instance_host>:<your_instance_port> <your_application_name_srv>
```

## Configure CAP for Local Access via SSH Tunnel

Create/update the file `.cdsrc-private.json` in the project root with the following configuration:

```json
{
    "requires": {
        "[hybrid]": {
            "db": {
                "kind": "postgres",
                "credentials": {
                    "hostname": "127.0.0.1",
                    "port": 63306,
                    "username": "9d9ee6aea4df",
                    "password": "<PASSWORD_IN_INSTANCE_CREDENTIALS_KEY>",
                    "dbname": "FnxTLFsXjsJi",
                    "sslcert": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----",
                    "sslrootcert": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----"
                },
                "pool": {
                    "acquireTimeoutMillis": 2000
                }
            }
        }
    }
}
```

## Running the CAP Application in Hybrid Profile

Once the `.cdsrc-private.json` file is in place and the SSH tunnel is active, start the CAP application locally using the hybrid profile:

```bash
cds w --profile hybrid
```

---

*Reading time: 2 minutes*
