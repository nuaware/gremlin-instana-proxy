# gremlin-instana-proxy
Kubernetes reverse proxy to enable Gremlin-Instana integration.

## Before you begin
We're assuming you have already set up instana and gremlin in Kubernetes (instana deployed as a DaemonSet) and they are both operational.


### Edit the NGINX conf file

Make sure you adjust the file `default.conf` to reflect your environment (hostnames, ssl, etc.)
 
### Create secrets and configMap

This deployment uses an *NGINX* container that listens on port 443 (https) only.
For this reason we need to put our ssl certficate chain and private key in a Kubernetes secret.	

```
$ kubectl create secret generic gremlin-instana-proxy-certs --from-file=ssl.key --from-file ssl.cert -n instana-agent
```

Deploy the *NGINX* configuration file to a configMap:

```
$ kubectl create configmap gremlin-instana-proxy-config --from-file default.conf -n instana-agent
```

Finally, create a `.htpasswd` file to set up basic auth in *NGINX*:

```
$ htpasswd -c .htpasswd api-user
New password:
Re-type new password:
Adding password for user api-user
```
Store this in a Kubernetes secret:

```
$ kubectl create secret generic gremlin-instana-proxy-credentials --from-file=.htpasswd -n instana-agent
```

Create a `base64` encoded string to autenticate the Gremlin webhook to *NGINX*:

```
$ echo -n api-user:<password> | base64
```

Store this string for use when configuring the Gremlin Webhook.

### Deploy the gremlin-instana-proxy

Our Kubernetes manifest creates three resources:

* `deployment.apps/gremlin-instana-proxy`: The actual NGINX proxy container.
* `service/gremlin-instana-proxy`: A LoadBalancer to expose the proxy to the outside world.
* `service/gremlin-instana-api`: A service to expose the instana agent host-API to the internal cluster. This is used by the proxy to connect to the API.

Deploy the manifest using the following command:

```
kubectl apply -f ./gremlin-instana-proxy.yaml -n instana-agent
```

### Create the Gremlin Webhook

In the Gremlin web console, click on the user icon in the top right, navigate to *Team Settings* and select the *Webhooks* tab.
Click on *New Webhook* and fill in the following fields:

| Key           | Valu          | 
| :------------ |:------------- | 
| Name.         | A name for your Webhook | 
| Description   | A description for your Webhook |
| Request URL   | https://*\<hostname>*.*\<domainname>*/api/com.instana.plugin.generic.event |
| Custom Header Key | Authorization |
| Custom Header Value | base64 encoded string |
| Request Method | POST |
| Request Format | JSON |

Select the events you wish to forward to Instana and add the JSON payload.
Information on what the Instana API accepts can be found [here](https://www.instana.com/docs/api/agent/#event-sdk-web-service).

An example payload:

```
{
  "title": "Gremlin attack: ${ATTACK_TYPE} Status ${STATUS}",
  "severity": "5",
  "text": "Team: ${TEAM_ID} Attack ID: ${ATTACK_ID} Attack Type:  ${ATTACK_TYPE} Status: ${STATUS}"
}
```

Fire of an attack and validate if you see an incoming event in Instana. To troubleshoot watch for the logs of the `gremlin-instana-proxy` pod.

A successful POST would look similar to this:

```
192.168.25.207 - api-user [11/Feb/2021:18:02:35 +0000] "POST /api/com.instana.plugin.generic.event HTTP/1.1" 204 0 "-" "-" "-"
```
