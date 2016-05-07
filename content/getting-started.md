+++
categories = ["general"]
date = "2016-05-05T11:57:51+09:00"
weight = 200
tags = ["document"]
title = "Getting Started"

+++

This guide shows you how to install and configure **fireap** to run it,
and what is needed beforehand.

## Prerequisites

### Software Requirements

- Consul v0.6.3 or later (recommended; v0.4.0 or higher is needed at least)
- Ruby v2.3.0 or later (recommended, but not necessarily)

### Set up Consul Cluster

Before you start with **fireap**, you need a _Consul_ cluster.  
Please refer to documentation on [consul.io](https://www.consul.io/docs/)
to set up _Consul_ cluster in your environment.

## Install

NOTE:

- Install ruby and gems environment for system user who will execute
**fireap** tasks.
- You need to install **fireap** and ruby environment on every machine where
**fireap** tasks will be executed.

```
git clone https://github.com/key-amb/fireap.git
cd fireap
# Install ruby gems in depend. Install "bundler" gem beforehand.
bundle install
bin/fireap help
```

If last command successfully shows the CLI usage, **fireap** is ready for you.

Instead HEAD of master branch, you can download released archives from
https://github.com/key-amb/fireap/releases . It is your choice.

## Configure

### Configure "fireap"

To run `fireap` commands, you need a configuration file whose format is
[TOML](https://github.com/toml-lang/toml).

The snippet below shows a sample configuration file:

<pre>
<code class="hljs yaml">## General Settings
# url = "http://localhost:8500"
# enable_debugging = "true" # For development Only

[log]
level  = "INFO"  # from Ruby Logger::LEVEL
rotate = "daily" # from Ruby Logger shift_age
# file = path/to/logfile

## Common Task Settings
# Can be overridden by each app settings
[task]
max_semaphores  = 5   # Max concurrency when one node can be "pulled" by others
wait_after_fire = 15  # seconds. Don't wait if not defined
watch_timeout   = 120 # seconds
on_command_failure = "abort" # or "ignore". Default is "abort"

# You can define common task commands for all apps here.
# Available variables for command formats:
#   - @app            ... Target App's Name
#   - @remote.name    ... Node Name in Consul Cluster
#   - @remote.address ... Node's Ipaddress in Consul Cluster
#   - ENV             ... Given Environment Vars for Watch Command
before_commands = [
    "echo Task <%= @app %> Started.",
]
# exec_commands = [] # Probably different for each apps.
after_commands = [
    "echo Task <%= @app %> Ended.",
]

# Belows are not implemented yet:
# command_timeout = 60

## Settings for each Task target Application
[task.apps.foo]
# max_semaphores  = 3
# before_commands = []
exec_commands = [
    #"scp -rp <%= @remote.name %>:<%= ENV['HOME'] %>/<%= @app %> <%= ENV['HOME'] %>/"
    "scp -rp <%= @remote.address %>:<%= ENV['HOME'] %>/<%= @app %> <%= ENV['HOME'] %>/"
]

# You can specify Consul service and tag to filter the task propagation targets.
# It can be normal string or regexp; if you want to specify regexp, use the keys
# "service_regexp" or "tag_regexp".
# If you miss "service" or "service_regexp" filter, "tag" and "tag_regexp" won't
# be evaluated.
service = "foo"
tag     = "v1"
# service_regexp = "^foo(:[a-z]+)?$"
# tag_regexp = "^v[0-9]$"

[task.apps.bar]
on_command_failure = "ignore"
before_commands = []
exec_commands = [
    "date '+%FT%T' > /tmp/bar.updated_at.txt",
    "rsync -az --delete --exclude='.git*' <%= @remote.address %>:<%= ENV['HOME'] %>/<%= @app %> <%= ENV['HOME'] %>/",
]
after_commands = []
</code></pre>

Some of configurations are commented out in this snippet.
You can enable it by uncommenting.

You find a sample in [config/sample.toml](https://github.com/key-amb/fireap/blob/master/config/sample.toml)
in this repository, which is similar to the one above.

Here are supplementary explanations:

- In `General` section:
  - You can change HTTP API address and port of _Consul agent_ by overwriting
`uri` parameter, which is called by **fireap** command.
- In `[task]` section:
  - You can configure common parameters through all tasks in this section.
  - Every parameter in this section can be overwritten in subordinate sections.
For example, if you want to change a parameter in task _"foo"_, you can overwrite
it in section `[task.apps.foo]`
- Commands combination:
  - Command list executed are combination of `before_commands`, `exec_commands`
and `after_commands`. Keep in mind that parameter overwriting takes place which
is explained above.

You can check your configuration by running `fireap task` command.  
For the sample configuration above, it goes as follows:

```bash
% fireap task -c config/sample.toml
== Configured Tasks ==
[Summary]
+-----+--------+---------+-----------+---------+------+---------------+
| App | MaxSem | Timeout | OnCmdFail | Service | Tag  | WaitAfterFire |
+-----+--------+---------+-----------+---------+------+---------------+
| foo | 5      | 120 sec | ABORT     | ^foo$   | ^v1$ | 15 sec        |
| bar | 5      | 120 sec | IGNORE    |         |      | 15 sec        |
+-----+--------+---------+-----------+---------+------+---------------+
[App "foo" - Commands]
+---+-----------------------------------------------------------------------------------+
| # | Command                                                                           |
+---+-----------------------------------------------------------------------------------+
| 1 | echo Task <%= @app %> Started.                                                    |
| 2 | scp -rp <%= @remote.address %>:<%= ENV['HOME'] %>/<%= @app %> <%= ENV['HOME'] %>/ |
| 3 | echo Task <%= @app %> Ended.                                                      |
+---+-----------------------------------------------------------------------------------+
[App "bar" - Commands]
+---+--------------------------------------------------------------------------------------------+
| # | Command                                                                                    |
+---+--------------------------------------------------------------------------------------------+
| 1 | date '+%FT%T' > /tmp/bar.updated_at.txt                                                    |
| 2 | rsync -az --delete --exclude='.git*' <%= @remote.address %>:<%= ENV['HOME'] %>/<%= @app %> |
|   |  <%= ENV['HOME'] %>/                                                                       |
+---+--------------------------------------------------------------------------------------------+
```

### Configure Consul Agent

Your Consul agent need to be passed `-watch` option by command line or `watches` parameter in your configuration file.  
Read [document on consul.io](https://www.consul.io/docs/agent/options.html) for details of agent configuration.

Here is an example of configuration file of Consul client agent:

```json
// consul-agent-conf.json
{
  "server":   false,
  "data_dir": "/opt/consul",
  "watches": [
    {
      "type": "event",
      "name": "FIREAP:TASK",
      "handler": "<FIREAP HANDLER COMMAND>"
    }
  ]
}
```

#### Watch Handler Command

`<FIREAP HANDLER COMMAND>` in the example configuration above is a watch handler in Consul.  
This can be any executable on your machine.

There can be two situations:

- CASE1) When system user who runs `consul agent` is the same to who runs `fireap`.
  - In this case, HANDLER COMMAND comes simple. Ex) `fireap reap -c /path/to/config.toml`
- CASE2) When system user who runs `consul agent` is different from who runs `fireap` command.
  - The former user must be able to switch to the latter user on `fireap` execution by any command such as `sudo` or `setuidgid`.
  - Suppose the latter user's name is `reaper`, an example of HANDLER COMMAND using `sudo` goes like this: `sudo -u reaper -- fireap reap -c /path/to/config.toml`

You can specify the configuration file path by shell environment varialbe
`$FIREAP_CONFIG_PATH` as well as command line option `-c|--config <PATH>`.

If you want to check what will happen preliminarily, you can add `--dry-run`
option to `fireap reap` command.

After tasks are fired, you can find the task execution sequence in the log
file on _subscriber_ nodes.

See [CLI Usage](../cli-usage) for more information of `fireap reap` command.

## Run "fireap" Task

Execute following steps on _publisher_ node to run **fireap** task in the
cluster:

1. Update materials on _publisher_ host which are needed for _subscribers_
to "pull" them from _publisher_.
1. Execute `fireap fire` command on _publisher_ host. It goes like below:

```sh
fireap fire -a <APP> -v <VERSION> -c /path/to/config.toml
```

`<APP>` is the task name in your **fireap** configuration file; `foo`
or `bar` can be specified in the sample configuration of this document.

`<VERSION>` is a ascii word to indicate _version_ of the task such as
"v1.0" or "ver-2.0.0-beta1".

See [CLI Usage](../cli-usage) for more information about this command.

## References

- [Watches - Consul by HashiCorp](https://www.consul.io/docs/agent/watches.html)
