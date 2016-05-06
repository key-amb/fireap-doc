+++
categories = ["general"]
date = "2016-05-05T11:59:09+09:00"
tags = ["document"]
title = "CLI Usage"

+++

This guide shows overview of `fireap` subcommands and typical usecases of
`fireap` CLI.

As for detailed usage of CLI, run `fireap help`.

## General Information

Global options:

- `-c|--config=/path/to/config.toml`
  - Option to specify path of configuration file.
  - When you set `$FIREAP_CONFIG_PATH` environment variable, you can omit this
option.

## Subcommands

### `fireap task`

```bash
fireap task [-w <WIDTH>] [OPTIONS]
```

This shows the task settings under `[task]` sections in the configuration file.  
By this command, you can also check if the configuration is valid or not.

### `fireap fire`

```bash
fireap fire -a <APP> [-v <VERSION>] [OPTIONS]
# To limit target node to execute the task
fireap fire -a <APP> --node-name|-n=<NODE_NAME> [-v <VERSION>] [OPTIONS]
```

This fires a **fireap** task propagation.

If you omit `-v|--version=<VERSION>` option, **fireap** start with version _"1"_
if it is the first execution.
When you have already executed the task, **fireap** increments the version
which is taken in the last execution.  
For example, when the version last executed is _"v1.0.0"_, next version will be
_"v1.0.1"_ without `-v|--version` option.

Read [Getting Started](../getting-started) for more information.

### `fireap clear`

```bash
fireap clear -a <APP> [OPTIONS]
```

`fireap fire` holds lock for each `<APP>` during the command execution to avoid
multiple execution of a task.

You do not need to run this in usual operation.  
But **fireap** supports this subcommand for rare possibility.

### `fireap reap`

```bash
fireap reap [OPTIONS]
fireap reap --dry-run [OPTIONS]
```

This receives contents of _Consul Event_ from STDIN, then executes configured
task.  
It is supposed to be used as _watch handler_ of _consul agent_.

However, if you want to check how this works preliminarily, you can do it by
command like followings:

```bash
curl http://localhost:8500/v1/event/list | fireap reap [--dry-run] [OPTIONS]
```

With `--dry-run` option, this does not execute commands set in the task but
prints them.

Read [Getting Started](../getting-started) for detailed instruction to use this
as _Consul Watch Handler_.

### `fireap monitor`

```bash
fireap monitor -a <APP> [OPTIONS]
fireap monitor -a <APP> --one-shot [OPTIONS]
```

This shows data in _Consul Kv_ related to `<APP>`.  
You can check task propagation by this.

Without `-o|--one-shot` option, it continuously shows the data until it
receives `INT` signal (corresponds to `Ctrl-C` in many terminal
environments).
