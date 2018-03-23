# Ansible Role: `rsyslog`

This Ansible role allows you to install `rsyslog` and manage its configuration.

For more information about `rsyslog`, please check [the official project page](https://www.rsyslog.com/).

This page should also help you understand the basics of rsyslog and thus the configuration options of this Ansible role: [Configuration : basic structure](https://www.rsyslog.com/doc/v8-stable/configuration/basic_structure.html).


**IMPORTANT**: this role should be able to manage the configuration for clients, relayers and/or central servers.


## Role variables

Variables and *properties* in bold are mandatory. Others are optional.

| Variable name                 | Description                                                                        | Default value        |
| ----------------------------- | ---------------------------------------------------------------------------------- | -------------------- |
| `rsyslog_additional_packages` | List of additional packages to install with rsyslog. (i.e. `rsyslog-imrelp`)       | `[]`                 |
| `rsyslog_working_dir`         | Path to the directory where rsyslog must store the queue files.                    | `/var/spool/rsyslog` |
| `rsyslog_tls`                 | A `rsyslog_tls` dict. See [rsyslog_tls properties](#rsyslog_tls-properties) below. | `{}`                 |
| `rsyslog_templates`           | A list of [template](#template-properties).                                        | `[]`                 |
| `rsyslog_rulesets`            | A list of [ruleset](#ruleset-properties).                                          | `[]`                 |
| `rsyslog_inputs`              | A list of [input](#input-properties).                                              | `[]`                 |
| `rsyslog_outputs`             | A list of [ouput](#output-properties).                                             | `[]`                 |

As you can see, the default configuration does nothing. It's just an empty shell.


### rsyslog_tls properties

`rsyslog_tls` is a `dict` that stores some paths to the needed certificates and keys needed for TLS to  work.

If you plan to use TLS (be it with `imtcp` or with `imrelp`), you **have to** specify all 3 properties.

| Property name | Description                                                                 |
| ------------- | --------------------------------------------------------------------------- |
| **`cacert`**  | Path to the CA certificate.                                                 |
| **`cert`**    | Path to the machine certificate (this certificate must be signed by the CA. |
| **`key`**     | Path to the private key corresponding to `rsyslog_tls.cert`.                |


### template properties

| Property name | Description                           |
| ------------- | ------------------------------------- |
| **`name`**    | Name of the template. Must be unique. |
| **`string`**  | Template.                             |

:heavy_exclamation_mark:

- For now we only support **string templates**. *list templates*, *subtree templates* and *plugin templates* are **not** supported. *options* aren't either.

:green_book: [Documentation](https://www.rsyslog.com/doc/v8-stable/configuration/templates.html)


### ruleset properties

| Property name | Description                                                                                                 |
| ------------- | ----------------------------------------------------------------------------------------------------------- |
| **`name`**    | Name of the ruleset.                                                                                        |
| **`script`**  | Instructions to execute when the ruleset is reached. Please see official documentation for further details. |

:green_book: [Ruleset documentation](https://www.rsyslog.com/doc/v8-stable/concepts/multi_ruleset.html)
:green_book: [RainerScript documentation](https://www.rsyslog.com/doc/v8-stable/rainerscript/index.html)

#### inputs properties

| Property name    | Description                                              |
| ---------------- | -------------------------------------------------------- |
| **`module`**     | Name of the module to load.                              |
| **`parameters`** | A dict of parameters passed when loading **the module.** |
| **`listeners`**  | A list of [listeners](#listeners-properties).            |

:heavy_exclamation_mark:

- Only modules that have at least one listener will be loaded. If you don't provide at least one listener, the module will be ignored.
- The `parameters` dict doesn't follow a strict, fixed schema. Keys are basically the names of the options supported by the module. Values must be set accordingly. If an option accept an array, you have to provide a list. The template will transform it into the expected array. Please also be aware that some modules have mandatory options. Please refer to the module documentation.
    
:green_book: [List of input modules](https://www.rsyslog.com/doc/v8-stable/configuration/modules/idx_input.html)

#### listener properties

A *listener* consists in a set of options **for the input**. It is represented as a `dict`.

A module can have multiple listeners defined with different options. For example, you may want to accept logs coming on UDP ports 541, 542 and 543 and apply a different ruleset in each case. In this particular example, you would have to define 3 different listeners for the same module :

```yaml
---

#[snip]

rsyslog_inputs:
  - module: imudp
    parameters: {}
    listeners:
      - port: 541
        ruleset: "UDP541"
      - port: 542
        ruleset: "UDP542"
      - port: 543
        ruleset: "UDP543"
...
```

Listener properties depends on the options supported by the module. So, keys are basically the names of the options supported by the module. Please note that some modules have mandatory options. Please refer to the module documentation.

We strongly advise to use **rulesets** to keep your configuration clean.

:green_book: [Documentation](https://www.rsyslog.com/doc/v8-stable/configuration/input.html)


### outputs properties

| Property name | Description                                       |
| ------------- | ------------------------------------------------- |
| **`module`**  | Name of the module to load.                       |
| **`actions`** | A list of [actions](#actions-element-properties). |

:green_book: [List of output modules](https://www.rsyslog.com/doc/v8-stable/configuration/modules/idx_output.html)

#### actions element properties

| Property name    | Description                              |
| ---------------- | ---------------------------------------- |
| **`selector`**   | *Selector* that catches the message.     |
| **`parameters`** | A dict of parameters **for the filter**. |

:heavy_exclamation_mark:

- The `parameters` dict doesn't follow a strict, fixed scheme. Keys are basically the names of the options supported by the module. Values must be set accordingly. If an option accepts an array, you have to provide a list. The template will transform it into the expected array. Please also be aware that some modules have mandatory options. Please refer to the module documentation.

:green_book: [Selector documentation](https://www.rsyslog.com/doc/v8-stable/configuration/sysklogd_format.html#selectors)


## Examples

### RELP server

In this first example, we want to setup a *loghost* that centralizes logs of several *clients*.

1. It accepts logs via the RELP protocol,
2. only over TLS,
3. on port 6514.


1. It outputs the received logs in a file,
2. that is specific for each client,
3. in RFC5424 format.

```yaml
---
rsyslog_additional_packages:
  # For TLS:
  - "rsyslog-gnutls"
  # For RELP:
  - "rsyslog-relp"
  # For SELinux:
  # CentOS:
  - "policycoreutils-python"
  # Debian:
  - "policycoreutils-python-utils"

rsyslog_working_dir: "/var/spool/rsyslog"

rsyslog_tls:
  cacert: "/etc/ssl/ca.cert"
  cert: "/etc/ssl/loghost.cert"
  key: "/etc/ssl/private/loghost.pem"

rsyslog_templates:
  - name: "fromRemote"
    string: "/var/log/remote/%fromhost%.log"
  - name: "rfc5424Format"
    string: "<%PRI%>%PROTOCOL-VERSION% %TIMESTAMP:::date-rfc3339% %HOSTNAME% %APP-NAME% %PROCID% %MSGID% %STRUCTURED-DATA% %msg%\\n"

rsyslog_rulesets:
  - name: "remote"
    script: |-4
            action(
                type="omfile"
                dynaFile="fromRemote"
                template="rfc5424Format"
            )
            stop

rsyslog_inputs:
  - module: imrelp
    parameters:
      tls: "on"
      tls.authmode: "name"
      tls.permittedpeer:
        - "10.0.0.100"
        - "10.0.0.101"
        - "10.0.0.102"
        - "*.localdomain"
    listeners:
      - port: 6514
        ruleset: "remote"

rsyslog_outputs: []
...
```

### RELP client

In this second example, we want to setup a *client* that forwards all its logs to the previously configured *loghost*.

1. It sends logs via the RELP protocol,
2. only over TLS,
3. on port 6514.

```yaml
---
rsyslog_additional_packages:
  # For TLS:
  - "rsyslog-gnutls"
  # For RELP:
  - "rsyslog-relp"

rsyslog_working_dir: "/var/spool/rsyslog"

rsyslog_tls:
  cacert: "/etc/ssl/ca.cert"    # MUST be the same as the one used on the loghost.
  cert: "/etc/ssl/client.cert"
  key: "/etc/ssl/private/client.pem"

rsyslog_templates: []
rsyslog_rulesets: []

rsyslog_outputs:
  - module: omrelp
    actions:
      - selector: "*.*"
        parameters:
          target: "loghost.localdomain"
          tls: "on"
          tls.cacert: "{{ rsyslog_tls.cacert }}"
          tls.mycert: "{{ rsyslog_tls.cert }}"
          tls.myprivkey: "{{ rsyslog_tls.key }}"
          tls.authmode: "name"
          tls.permittedpeer:
            - "loghost.localdomain"
...
```

## Contributing


