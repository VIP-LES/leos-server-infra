# Dev Branch Changes

This document summarizes what has been implemented on the `dev` branch of
`leos-server-infra` relative to `master`.

The focus of this branch is to make infrastructure automation match the new
split ground-station architecture:

- Raspberry Pi forwards telemetry to a lab host
- lab host runs ingest, database, and dashboards

## High-Level Result

This branch adds deployment automation for the reworked
`leos-S26-ground-station` `dev` branch.

Compared to `master`, the infra repo no longer only provisions generic hosts.
It now also deploys the ground-station app in role-specific ways:

- lab host deployment
- Pi forwarding deployment
- Tailscale install and join automation for both host types

## Major Infrastructure Changes

### 1. New Lab Deployment Role

A new role has been added:

- `server/roles/groundstation_lab`

This role is now included from `server/site.yml`.

Its job is to:

- clone `leos-S26-ground-station`
- check out the `dev` branch
- create a Python virtualenv
- install Python dependencies
- template a lab-host `.env`
- run `docker-compose up -d`
- install and start `leos-ingest.service`

This is the first time the infra repo has explicitly automated deployment of the
ground-station application stack rather than only preparing the host OS.

### 2. Pi Bootstrap Updated for Forwarding Mode

`rpi-groundstation/bootstrap.yml` now does more than clone the app and install
dependencies.

It now also:

- checks out the `dev` branch of `leos-S26-ground-station`
- creates a persistent spool directory
- templates a Pi-specific `.env`
- installs `leos-forwarder.service`
- enables and starts the forwarder service

This aligns the Pi provisioning path with the new architecture where the Pi is a
queue + forwarder host instead of a full local ingest stack.

### 3. Host-Specific Environment Templating

New templates were added for:

- lab host `.env`
- Pi host `.env`
- `leos-ingest.service`
- `leos-forwarder.service`

This gives the two hosts separate runtime identities:

- lab host runs `GROUNDSTATION_MODE=lab`
- Pi runs `GROUNDSTATION_MODE=pi`

### 4. Tailscale Is Now Actually Wired Into Automation

This branch now installs and starts Tailscale on:

- the lab host through `server/site.yml`
- the Pi through `rpi-groundstation/bootstrap.yml`

Both paths support:

- `tailscale_authkey`
- `tailscale_hostname`
- `tailscale_ssh`

The Pi env template now also supports:

- `tailscale_lab_host`

which is used as the default host part of the Pi `INGEST_URL`.

For test deployments, the Pi path also supports:

- `groundstation_fake_radio_enabled`
- `groundstation_fake_radio_input_path`
- `groundstation_fake_radio_repeat`
- `groundstation_fake_radio_interval_s`

When enabled, the Pi deployment can point at the built-in fake telemetry input
fixture from the application repo.

Important operational note:

- first-boot bootstrap and ongoing configuration are still slightly different concerns
- when a brand-new Pi may not be reachable reliably over local networking,
  especially on Apple hotspot, Raspberry Pi cloud-init / `user-data` is still a
  useful first-boot path to get Tailscale online before Ansible runs
- after that first reachability problem is solved, this repo's Ansible
  automation is intended to own the ongoing Tailscale setup and application
  deployment

## Important Wiring Assumptions

This branch assumes:

- the `dev` branch of `leos-S26-ground-station` exists remotely
- Tailscale auth/config values are supplied by the operator
- the lab host can be reached from the Pi over the configured Tailscale hostname
- the Pi points `INGEST_URL` at that lab host

By default the Pi env template points at:

- `http://{{ tailscale_lab_host }}:4000`

That host value must match the actual Tailscale hostname or MagicDNS name of the
lab machine.

## Ansible Fixes Included During This Branch

Two important deployment issues were corrected while implementing this branch:

- the lab deployment role now runs after `std-packages` so Git is present before
  the role tries to clone the app repo
- the lab virtualenv and pip install steps now run as the lab app user instead
  of as `root`, avoiding ownership mismatches with the systemd service user

## Relationship to `leos-S26-ground-station`

This infra branch is designed specifically around the corresponding
`leos-S26-ground-station` `dev` branch, which now contains:

- the Pi/lab mode split
- the renamed ingest API
- two hypertables
- the SQLite queue
- the forwarder
- the fake/replay telemetry path

Without those application changes, the deployment changes in this repo do not
fully make sense on their own.

## Still Not Fully Wired

These are the main remaining assumptions that are not yet fully automated:

- the real Pi `radio_receiver.py` does not exist yet, so Pi mode is currently a
  forwarder plus optional fake/replay source
- the very first Pi bootstrapping step may still require cloud-init assistance
  when local networking conditions make initial SSH access unreliable

This branch now includes a server-side secrets example file:

- `server/vars/secrets.example.yml`

Operators should copy it to `server/vars/secrets.yml` with real values before
running the lab host playbook.
