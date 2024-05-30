# sheploy

Shell deploy - KISS, multi-stage deployment suitable for docker swarm.

- Tasks are simple bash scripts, that will get executed like this:
```sh
cat $command_path | ssh -T $host /bin/bash $VERBOSE_FLAG $REMOTE_DEBUG_FLAG
```
- It is supposed ssh connection is tested before running `sheploy`.
- The `sheploy` command supports `--dry-run` (shoert `-n`), and it is advised to use it.
- The deployments are expected to be in `~/deploys` but it can be changed using `--deploys-dir` option. See `sheploy --help` for other options.
- In the [deploys](deploys) directory is an example configuration, neither the layout, nor the tasks scripts are supposed to be used out-of-the-box. This is not a cookbook of receipts.


### Installation

Install only on management node:

```sh
git clone git@github.com:brablc/sheploy /usr/local/lib/sheploy
ln -s /usr/local/lib/sheploy/bin/sheploy /usr/local/sbin/
```

### Example deploys

The example is demonstrated using:

- one manager node (`swarm-mng`),
- two worker nodes (`swarm-wrk1`, `swarm-wrk2`),
- two roles `any` used by all hosts and `wrk` used by worker nodes.

Please watch the comments:

```
▼ deploys                         # base dir with deploys
  ▼ hosts                         # list of hosts accessible via ssh command (resolvable or using ~/.ssh/config)
    ▼ swarm-mng
      ▼ files/etc                 # each host can have its own `files` dir or it can symlink role
        ▼ cron.d
          - backup.example
        ▼ postfix
          - generic.example
      ▼ tasks
        ~ 090-lazydocker
        ~ 090-lazygit
        ~ 090-postfix-generic
      ▶ tasks-any                 # to use role's task symlink the folder, keep `tasks-` prefix
    ▼ swarm-wrk1
      ▶ files-wrk
      ▶ tasks-any
      ▶ tasks-wrk
  ▼ roles                         # if you have multiple hosts with same deployment use roles
    ▼ any/tasks                   # host can have multiple roles
      ~ 010-apt-update
      ~ 011-apt-install
      ~ 020-postfix
      ~ 020-sysstat
      ~ 020-zellij
    ▼ wrk/tasks
      ~ 030-postfix-relay
  ▼ tasks                         # all the tasks above can be symlinked to a common folder or they may have script directly
    - apt-install
    - apt-update
    - postfix
    - postfix-relay
    - sysstat
    - zellij
```

> [!TIP]
> The host name should actually be alias from [~/.ssh/config](https://man7.org/linux/man-pages/man5/ssh_config.5.html).
> You may use `backup@swarm-wrk1` as host name to connect as different user to the same machine.

> [!TIP]
> Watch the numeric prefixes of tasks, they can be used to execute tasks in stages, or to run specific task.
> Tasks from host and its roles are merged and sorted by name and executed in that order.

Having the `deploys` directory ready, this is the recommended way of running:

```sh
# Deploy files, that are different on other hosts, owner and permissions are kept
sheploy --dry-run --config-type=files

# prepare all hosts - run tasks starting with 0
sheploy --dry-run --prefix=0

# additional phases - like copying libraries gathered in step 0 on manager host
sheploy --dry-run --prefix=1

# install new packages only on worker nodes
sheploy --dry-run --host-prefix=cb-wrk3 --prefix=011

# create a new worker host - would run full suit on one node
rsync deploys/hosts/{example-wrk1,example-wrk3}/
sheploy --dry-run --host-prefix=cb-wrk3
```
