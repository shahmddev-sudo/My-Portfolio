---
title: "The Ansible Playbook That Worked Everywhere Except the One Place It Mattered"
description: "A playbook that provisioned three environments flawlessly failed on a fresh VM because of a task that silently succeeded. What I learned about idempotency the hard way."
pubDate: 2021-08-14
tags: [ansible, devops, iac]
draft: false
---

Every infrastructure tutorial shows you the happy path: write the playbook, run it, everything works. Nobody shows you the failure mode where the playbook **reports success** and the server is still broken.

That's exactly what happened to me the first time I tried to provision a clean Ubuntu VM.

## The setup

I had a playbook that installed Docker, created a service user, deployed a compose file, and started the stack. It had provisioned two environments already — I ran it a third time against a fresh VM and the deploy "succeeded." Green checkmarks, zero changed tasks, exit code zero.

The stack wasn't running.

## What actually happened

One task in the middle of the playbook looked like this:

```yaml
- name: Ensure the compose project directory exists
  file:
    path: /opt/myapp
    state: directory
```

Run it on a machine where that path already existed as a **file** (left over from a bad manual experiment during provisioning) and the task doesn't fail — it reports `ok`, because from Ansible's point of view the path exists. State name says directory; module checks existence. The mismatch never surfaces.

Then `docker compose up` ran from the wrong working directory, found nothing, and the follow-up health check task — which I had written with a generous retry loop — politely retried itself into a false positive because I checked "is the port listening" against the *host*, not the container.

## The fix, and the habit it created

Two changes:

```yaml
- name: Ensure the compose project directory exists
  file:
    path: /opt/myapp
    state: directory
  register: appdir

- name: Fail fast if the path is not a directory
  assert:
    that:
      - appdir.state == 'directory'
```

And more importantly: health checks should assert the thing you actually care about ("the API answers 200 with the right body"), not a proxy for it ("something listens on the port"). A port can be held open by anything. An HTTP check with an expected payload can't.

## What I'd tell past me

Idempotency isn't just "running twice is safe." It's "running against any state reports the truth." The moment you have a task that says `ok` when reality disagrees with your declaration, you've built a playbook that works everywhere except the one place it matters — the machine you're actually provisioning.
