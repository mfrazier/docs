+++
date = "2016-07-03T04:02:20Z"
title = "Components And Containers"
description = "The `components` section of the Replicated YAML defines how the containers will be created and started."
weight = "206"
categories = [ "Packaging" ]

[menu.main]
Name       = "Components And Containers"
identifier = "components-and-containers"
parent     = "/packaging-an-application"
url        = "/docs/packaging-an-application/components-and-containers"
+++

The `components` section of the YAML defines how the containers will be created and started. A component is
a group of one or more containers that are guaranteed to run on the same node.

## Image Registry Location
For each container we must supply some basic information. First the source from which we will pull this image. This can be either a third
party private or public registry (ie Docker Hub Registry, Quay.io, or your own public registry) or Replicated's private vendor registry
(`replicated`). We must then specify the name of the image (`image_name`) and the tag (`version`).

```yml
components:
- name: DB
  containers:
  - source: public
    image_name: redis
    version: latest
    when: '{{repl ConfigOptionEquals "use_external_db" "0" }}'
    ...
```

When including a public image please use a valid image name. This can be any format that the Docker client is able to pull, including:

- image (trusted image from Docker Hub)
- namespace/image (public image from Docker Hub)
- host/namespace/image (public image from a different host)

For Replicated private images (when `source` = `replicated`, do not include the namespace in the `image_name`. The `image_name` field should only
include the image name, not the hostname or namespace. The `when` option allows container to be started conditionally.

## Privileged
In some advanced scenarios, we may need to run our container in privileged mode or specify a hostname to be set in the container. This is possible in the YAML by adding a couple of optional tags.

```yml
  - source: public
    image_name: redis
    ...
    privileged: true
    hostname: host01
```

## Ephemeral
Sometimes we may want a container that is not meant to continue running for the lifetime of the application. In this case we can mark that container as ephemeral.

```yml
  - source: replicated
    image_name: startup
    ...
    ephemeral: true
```

## Container Resource Constraints
Replicated supports the following constraint parameters:

- `cpu_shares` How much CPU time is allotted to the container details.
- `memory_limit` How much memory is allotted to the container details.
- `memory_swap_limit` How large a memory swap is allocated to the container details.

*note: Replicated requires you to specify entire byte count as shown below*

```yml
  - source: replicated
    image_name: redis
    ...
    cpu_shares: "1024"
    memory_limit: "8000000000" #8GB
    memory_swap_limit: "1000000000" #1GB
```

## CMD
Next we can optionally define a container CMD to execute when running our container.

```yml
  - source: public
    image_name: redis
    ...
    cmd: '["redis-server", "--appendonly", "yes"]'
```

## Config Files
The next section contains inline configuration files that we can supply to our container. Replicated will create a file within the
container with the specified path (filename) and contents. You may optionally specify an octal permissions mode as a string (file_mode),
and/or the numeric uid of a user & group (file_owner) to be applied to the resulting file in the container.

```yml
    config_files:
    - filename: /elasticsearch/config/elasticsearch.yml
      file_mode: "0644"
      file_owner: "100"
      contents: |
        path:
          data: /data/data
          logs: /data/log
          plugins: /elasticsearch/plugins
          work: /data/work
        http.cors.enabled: true
        http.cors.allow-origin: /https?:\/\/{{repl ConfigOption "hostname" }}(:[0-9]+)?/
```

## GitHub Reference
It is also possible to specify a file as a Github Reference, where the ref is the SHA of the commit. This ref will need to be updated any
time the file changes (we cache the remote file to remove this external dependency from the install time processes). The repository will
either need to be public or you will need to connect your Github account via the App Settings link of the Replicated Vendor Portal.
(Supports config files only)

```yml
    config_files:
    - filename: /elasticsearch/config/elasticsearch.yml
      source: github
      owner: getelk
      repo: elasticsearch
      path: files/elasticsearch.yml
      ref: c3636b396b2df172926816be5660c9cabc8c5355
      file_mode: "0644"
      file_owner: "100"
```

## Customer Files
It can also be helpful to request a customer supplied file. This file can be referenced by the name parameter and will be created within
the container at the specified path (filename).

```yml
    customer_files:
    - name: logstash_input_lumberjack_cert_file
      filename: /opt/certs/logstash-forwarder.crt
      file_mode: "0600"
      file_owner: "0"
    - name: logstash_input_lumberjack_key_file
      filename: /opt/certs/logstash-forwarder.key
      file_mode: "0600"
      file_owner: "0"
```

## Environment Variables
Next we have the option of specifying environment variables. There is also a flag provided to exclude anything secret from the support bundle.

```yml
  env_vars:
    - name: AWS_ACCESS_KEY_ID
      static_val: '{{repl ConfigOption "logstash_input_sqs_aws_access_key" }}'
      is_excluded_from_support: true
    - name: AWS_SECRET_ACCESS_KEY
      static_val: '{{repl ConfigOption "logstash_input_sqs_aws_secret_key" }}'
      is_excluded_from_support: true
```

## Ports
We can use the ports property to publish a container's exposed port (private_port) to the host (public_port). The when property allows us to
conditionally expose that port mapping when some prior condition is satisfied. Use the interface property to force the public port to be
bound on a specific network interface.

```yml
    ports:
    - private_port: "80"
      public_port: "80"
      port_type: tcp
      when: '{{repl ConfigOptionEquals "http_enabled" "1" }}'
    - private_port: "443"
      public_port: "443"
      interface: eth0
      port_type: tcp
      when: '{{repl ConfigOptionEquals "https_enabled" "1" }}'
```

## Volumes
We can also specify volumes that will be bind mounted from our host to our container.
Optional properties:

- `permission` should be a octal permission string
- `owner` should be the uid of the user inside the container
- `is_ephemeral` Ephemeral volumes do not prevent containers from being re-allocated across nodes. Ephemeral volumes will also be excluded from snapshots. *supported as of 2.3.5
- `is_excluded_from_backup` exclude this volume from backup if Snapshots enabled

```yml
    volumes:
    - host_path: /data
      container_path: /data
      permission: "0644"
      owner: "100"
      is_ephemeral: false
      is_excluded_from_backup: true
```

## Logs
We can configure logs for containers by specifying the max number of logs files and the max size of the log files. The max size string should include
the size, k for kilobytes, m for megabytes or g for gigabytes. Log settings at the component level are inherited by the container and will be
used unless overwritten.

```yml
components:
  - name: sample-agent
    logs:
      max_size: 200k
      max_files: 2
    containers:
      - source: replicated
        logs:
          max_size: 500k
          max_files: 5
```

## Custom Support Bundle Files
And finally we have the option to specify additional files as part of the On-Prem Support Bundle. We can include both files from within our app's containers,
as well as output of commands executed within the container.
*Note: Every individual command will have 30 seconds to execute and then subsequently timeout.*

```yml
    support_files:
    - filename: /var/log/nginx/access.log
    - filename: /var/log/nginx/error.log
    support_commands:
    - filename: access_last_1000.log
      command: [tail, -n1000, /var/log/nginx/access.log]
    - filename: error_last_1000.log
      command: [tail, -n1000, /var/log/nginx/error.log]
```

## Restart Policies
Optionally, containers can be configured to be restarted automatically. Currently supported restart policies match those supported natively by Docker.
If the policy is not specified, the container will never be restarted. This behavior is equivalent to this setting:

### Never restart
```yml
  restart:
    policy: never
```
Specifying the following policy will always restart the container regardless of the exit code.

### Always restart
```yml
  restart:
    policy: always
```
Specifying the following policy will cause the container to be restarted with it terminates with an error. The max parameter is optional. If omitted, the
container will be restarted indefinitely.

### Restart on error only
```yml
  restart:
    policy: on-failure
    max: 1000
```
Please refer to our Examples page for additional component configuration examples.



## Config Files
Your application may have config files that require dynamic values. These values may be input by the person installing the software, values
specific to the environment your application is running in, values created by other containers or read from the embedded license via the
License API. To accomplish this, Replicated allows templatizing of its config values using the Go template language with a repl escape
sequence. When your application runs, Replicated will process the templates, and you have full access to the the Replicated template library.

## Customer Files
Sometimes it can be helpful to allow a customer to supply a file to your app. A good example of this is when your customer should supply an
SSL certificate and private key. To add customer supplied files to your container, you must first define the file as a config option, and
then create a link to it in any container that needs that file. Replicated will prompt for the file to be uploaded and will ensure that the
file is at the correct location in the container when it is started.

## Support Files
When your customer visits the support page in their On-Prem installation, Replicated will automatically download a extensive set of files
from each host and container running your application. If your application creates any files that would be useful for remote troubleshooting,
list them here. You simply provide a path to the file in the container, and when the support pack is downloaded, Replicated will include
your support files, if present. More detail here: Support Bundle.

## Environment Variables
The 12-factor app encourages the use of environment variables for configuration, and Replicated supports this design pattern. You can specify
environment variables, which will be injected into a container when it's created.

Environment variables can be created with static values or customer supplied values.

Environment variables support the Replicated template library.

## Exposed Ports
All ports listed in the Dockerfile with the EXPOSE directive will be automatically exposed when started. The Docker runtime will choose a
random port, ensuring that there are no conflicts. If you need to specify a specific public (host) port, you can list it here.

Common examples of when it is necessary to list an exposed port are for web server containers, or servers which have clients that are incapable
of discovering dynamic port mappings.

Port mappings support the Replicated template library.

## Volumes
Volumes are required for any persistent data created by your application. If you have data in a container that needs to be available to new
versions of your app, or data that should be backed up, define a volume to store it.

You need to specify only the container path of the volume. Replicated will pick an appropriate host path when the volume is first created. When
new versions of your container are deployed, the volume will be mounted in the updated container.

## Startup
The startup section of a container allows you to specify the CMD value that will be passed to your container when it's started. It's generally
good to end your Dockerfile with an ENTRYPOINT command. If you specify a value for the CMD, it will be passed as parameters to the your ENTRYPOINT.

As with all inputs to containers, you have full access to the Replicated template library when creating a CMD value.
