Privilege Escalation Fun with Docker

[Docker](), if you aren't already familiar with it, is a lightweight runtime and packaging tool. It's very similar to simply running a basic virtual machine, but with much less overhead. It's extremely nice for deploying applications as you can guarentee that they will run in identicle environments, and the commit-like image system is handy as well.

If you happen to have gotten access to a user-account on a machine, and that user is a member of the `docker` group, running the following command will give you a root shell.

<!-- Content Breaker -->

```bash
```

This looks like the following when you run it.

```bash
```

## The Problem

When you've spent any amount of time with Docker, a common complaint is that all of the commands require `sudo` prefixing them. The Docker daemon runs on the system itself, and you can interact with it directly through.


## The Solution
