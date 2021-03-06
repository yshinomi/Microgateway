## Microgateway on OpenShift: configure, install, upgrade, scale and more

* [Prerequisites](#prerequisites)
* [Deployment diagram](#diagram)
* [Operation commands](#ops-commands)
  * [Configure](#configure)
  * [Install](#install)
  * [Update strategies](#upgrade)
  * [Scale up/down](#scale)
  * [Autoscaling](#autoscaling)
  * [Logs](#logs)
  * [Health Check](#health-check)
  * [Uninstall](#uninstall)

### Prerequisites <a name="prerequisites"></a>
- An account on an OpenShift deployment
- The OpenShift Origin CLI
(https://docs.openshift.org/latest/cli_reference/get_started_cli.html) to
operate the Microgateway on OpenShift

### Deployment diagram <a name="diagram"></a>

[microgateway-on-openshift]: img/openshift_draw.io.png "Microgateway on OpenShift"
![alt text][microgateway-on-openshift]

The Microgateway cluster running on OpenShift is at least composed of:
- an OpenShift route exposing the Microgateway to users
- an OpenShift service load balancing requests to the Microgateway containers
- OpenShift pods hosting respectively a Microgateway container

Microgateway containers synchronize exposed API definitions with a database or a
key/value store.

*Note: The database/KV store and microservices can optionally run in the same
OpenShift*

### Operation commands <a name="ops-commands"></a>

OpenShift YAML file used in this document: [microgateway.yaml](../../../openshift/microgateway.yaml)

#### Configure <a name="configure"></a>

*Note: please refer to the main documentation for the list of required and optional
environment variables: https://docops.ca.com/ca-microgateway/1-0/EN.*

#### Install <a name="install"></a>

```
oc create --filename microgateway.yaml
```

With `microgateway.yaml` containing the definition of each OpenShift elements
including Microgateway pod hosting the container.

#### Update <a name="upgrade"></a>

Multiple update strategies are available:
- Rolling Strategy and Canary Deployments
- Recreate Strategy
- Custom Strategy
- Blue-Green Deployment using routes
- A/B Deployment and canary deployments using routes
- One Service, Multiple Deployment Configurations

Example of a rolling update configuration:
```
apiVersion: v1
kind: DeploymentConfig
metadata:
  name: microgateway-dc
spec:
  replicas: 3
  template:
    metadata:
      labels:
        name: microgateway
    spec:
      strategy:
        type: Rolling
        rollingParams:
          updatePeriodSeconds: 1
          intervalSeconds: 1
          timeoutSeconds: 120
          maxSurge: "20%"
          maxUnavailable: "10%"
          pre: {}
          post: {}
      containers:
      - name: microgateway
        image: caapimcollab/microgateway:beta2
        env:
          [...]
```

Details about deployment strategies can be found at
https://docs.openshift.com/container-platform/latest/dev_guide/deployments/deployment_strategies.html

#### Scale up/down <a name="scale"></a>

- Manual scaling:
  - Using the OpenShift YAML file (e.g. `microgateway.yaml`)

  The `replicas` key of the Deployment configuration block sets the number of
  Microgateway pods to deploy:

  ```
  apiVersion: v1
  kind: DeploymentConfig
  metadata:
    name: microgateway-dc
  spec:
    replicas: 3
  ```

  Then push the new configuration:
  ```
  oc apply --filename microgateway.yaml
  ```

  - Using the OpenShift command line "oc":

  In the previous example, the deployment configuration is named `microgateway-dc`.
  Instead of pushing a new deployment configuration, the `oc scale` command can
  be used:

  ```
  oc scale dc microgateway-dc --replicas=3
  ```

- Autoscaling:

  Autoscaling is done by adding an OpenShift element `HorizontalPodAutoscaler` to
  the OpenShift YAML file (e.g. `microgateway.yaml`).

  The following example scales up the OpenShift Deployment Configuration
  `microgateway-dc` if the CPU reaches 80%.

  ```
  apiVersion: extensions/v1beta1
  kind: HorizontalPodAutoscaler
  metadata:
    name: frontend
  spec:
    scaleRef:
      kind: DeploymentConfig
      name: microgateway-dc
      apiVersion: v1
      subresource: scale
    minReplicas: 1
    maxReplicas: 10
    cpuUtilization:
      targetPercentage: 80
  ```

  Then push the new configuration:
  ```
  oc apply --filename microgateway.yaml
  ```

  Details about autoscaling can be found at
  https://docs.openshift.com/container-platform/latest/dev_guide/pod_autoscaling.html

#### Logs <a name="logs"></a>

- Print logs:

```
oc logs --follow dc/microgateway-dc
```
Where `microgateway-dc` is the name of our Deployment Configuration defined
in `microgateway.yaml`.

#### Health check <a name="health-check"></a>

```
apiVersion: v1
kind: DeploymentConfig
metadata:
  name: microgateway-dc
spec:
containers:
- name: microgateway
  image: caapimcollab/microgateway:latest
  env:
    [...]

  ports:
    [...]

  readinessProbe:
    exec:
      command:
      - /opt/docker/rc.d/diagnostic/health_check.sh
    initialDelaySeconds: 180
    periodSeconds: 5
    timeoutSeconds: 5

  livenessProbe:
    exec:
      command:
      - /opt/docker/rc.d/diagnostic/health_check.sh
    initialDelaySeconds: 180
    periodSeconds: 5
    timeoutSeconds: 5
```
Where:
  - `microgateway-dc` is the name of our Deployment Configuration defined in `microgateway.yaml
  - `readinessProbe` determines if a container is ready to service requests
  - `livenessProbe` determines if a container is running properly to serve requests
  - `/opt/docker/rc.d/diagnostic/health_check.sh` is the Microgateway health check script running inside the container

Details about OpenShift Application Health Check can be found at
https://docs.openshift.com/container-platform/latest/dev_guide/application_health.html  

#### Uninstall <a name="uninstall"></a>

```
oc delete --filename microgateway.yaml
```
