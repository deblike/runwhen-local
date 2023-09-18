# Stuck? Read this

Not everything runs perfectly the first time. This page outlines some common issues and approaches to help get unstuck.&#x20;

{% hint style="info" %}
Please remember, you are always encouraged to reach out  to us on [Slack](https://runwhen.slack.com/join/shared\_invite/zt-1l7t3tdzl-IzB8gXDsWtHkT8C5nufm2A),  [GitHub](https://github.com/runwhen-contrib/runwhen-local) or [Discord](https://discord.com/invite/Ut7Ws4rm8Q)
{% endhint %}

### Couldn't Connect to localhost:7687

Saw this message?

```
Discovering resources...
Error 500 from Workspace Builder service for command "run": Couldn't connect to localhost:7687 (resolved to ()):
Failed to establish connection to ResolvedIPv6Address(('::1', 7687, 0, 0)) (reason [Errno 111] Connection refused)
Failed to establish connection to ResolvedIPv4Address(('127.0.0.1', 7687)) (reason [Errno 111] Connection refused)
```

Wait a few seconds and try again. A couple of the internal services are still starting up.&#x20;

### Nothing Happened?

If you've executed `run.sh` as indicated [here](getting-started/running-locally.md#generating-your-kubeconfig), but nothing happens or you get a generic `500`, check the container logs for additional details.&#x20;

```
$ docker exec -w /workspace-builder -- RunWhenLocal ./run.sh
Error 500 from Workspace Builder service for command "run": None
```

#### Getting Logs while Running Locally

In order to get more insights into the root cause of the issue, you can attach to the container image to gain further details:&#x20;

```
$ docker attach RunWhenLocal 
Directory /shared/output has been created.
Starting up neo4j
Waiting a bit before starting prodgraph REST server
WARNING  -  Config value 'build': Unrecognised configuration name: build
WARNING  -  Config value 'dev_addr': The use of the IP address '0.0.0.0' suggests a production environment or the use of a proxy to connect to the MkDocs server. However, the MkDocs' server is intended for local development purposes only. Please use a third party production-ready server instead.
INFO     -  Building documentation...
INFO     -  Cleaning site directory
INFO     -  Documentation built in 1.48 seconds
INFO     -  [13:28:09] Watching paths for changes: 'cmd-assist-docs/docs', 'cmd-assist-docs/mkdocs.yml'
INFO     -  [13:28:09] Serving on http://0.0.0.0:8081/
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions
Running migrations:
...
Starting prodgraph REST server
```

* In many cases, you may just see the following generic error, which is often related to [#authentication-issues](stuck-read-this.md#authentication-issues "mention")

```
Resetting neo4j models
Internal Server Error: /run/
[26/Jul/2023 22:30:03] "POST /run/ HTTP/1.1" 500 9619
```

####

#### Getting Logs while Running in Kubernetes

As with any pod in Kubernetes, you can obtain the logs with the `kubectl` command:&#x20;

```
$ kubectl logs -f [pod-name] -n [namespace]

```



### Authentication Issues

If you are running into authentication issues, there are a couple of items to check:&#x20;

* Check that the `kubeconfig` you generated is accessible inside of the container

```
$ docker exec -- RunWhenLocal cat /shared/kubeconfig
apiVersion: v1
clusters:
  - cluster: ...

```

* If the `kubeconfig` exists inside of the container, check that the kubeconfig works

```
$ KUBECONFIG=$workdir/shared/kubeconfig kubectl get ns
... 
```



* If the kubeconfig doesn't work,  check if you using a short-lived token, such as that generated by an auth plugin, that might be expired? If so, re-run the script used to create the `kubeconfig` as outlined in [#generating-your-kubeconfig](getting-started/running-locally.md#generating-your-kubeconfig "mention") and check the `kubeconfig` again

```
$ KUBECONFIG=$workdir/shared/kubeconfig kubectl get ns
I0726 18:27:23.879795   59905 versioner.go:58] the server has asked for the client to provide credentials
error: You must be logged in to the server (the server has asked for the client to provide credentials)

$ ./gen_rw_kubeconfig.sh
Directory '/Users/sheastewart/runwhen-local/shared' exists.
...
...

$ KUBECONFIG=$workdir/shared/kubeconfig kubectl get ns
NAME                STATUS   AGE
argo                Active   89d
artifactory         Active   75d
awx                 Active   120d
...
```

* If the above worked, then you should be able to successfully execute the `run.sh` command again

```
$ docker exec -w /workspace-builder -- RunWhenLocal ./run.sh
```



### Network Issues

In some cases, the configuration of your container runtime (e.g. docker/podman) might cause network issues. When attaching to the container, it might look like this:&#x20;

```
Retrying (Retry(total=2, connect=None, read=None, redirect=None, status=None)) after connection broken by 'NewConnectionError('<urllib3.connection.HTTPSConnection object at 0x7f03b1d11280>: Failed to establish a new connection: [Errno -3] Temporary failure in name resolution')': /apis/
Retrying (Retry(total=1, connect=None, read=None, redirect=None, status=None)) after connection broken by 'NewConnectionError('<urllib3.connection.HTTPSConnection object at 0x7f03b1d118b0>: Failed to establish a new connection: [Errno -3] Temporary failure in name resolution')': /apis/
Retrying (Retry(total=0, connect=None, read=None, redirect=None, status=None)) after connection broken by 'NewConnectionError('<urllib3.connection.HTTPSConnection object at 0x7f03b1b15a00>: Failed to establish a new connection: [Errno -3] Temporary failure in name resolution')': /apis/
Internal Server Error: /run/
[21/Jul/2023 18:53:46] "POST /run/ HTTP/1.1" 500 11703
```



#### Check Network Connectivity

* Test DNS resolution to the internet:&#x20;

```
$ docker exec -- RunWhenLocal wget www.google.com
--2023-07-27 11:19:16--  http://www.google.com/
Resolving www.google.com (www.google.com)... 142.251.32.68, 2607:f8b0:400b:80f::2004
Connecting to www.google.com (www.google.com)|142.251.32.68|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: unspecified [text/html]
Saving to: 'index.html.1'

     0K .......... ........                                     294K=0.06s

2023-07-27 11:19:16 (294 KB/s) - 'index.html.1' saved [18722]
```

* Test access to your cluster API endpoint (this is found inside of your kubeconfig)

```
docker exec -- RunWhenLocal wget https://[cluster api endpoint]
--2023-07-27 11:20:14--  https://[cluster api endpoint]
Connecting to [cluster api endpoint]... connected.
ERROR: The certificate of '[cluster api endpoint]' is not trusted # This is expected and normal
ERROR: The certificate of '[cluster api endpoint]' doesn't have a known issuer. # This is expected and normal