# Kraft-with-SASL-PLAIN

Deploy Confluent Platform With Kraft and SASL/PLAIN
================================

In this workflow scenario, you'll set up secure Confluent Platform clusters with
SASL PLAIN authentication, no authorization, and no inter-component TLS.

## Prerequisites

The following prerequisites are assumed for each scenario workflow:

* A Kubernetes cluster - any CNCF conformant version
* Helm 3 installed on your local machine
* Kubectl installed on your local machine
* A namespace created in the Kubernetes cluster - `confluent`
* Kubectl configured to target the `confluent` namespace:
  ```
  kubectl config set-context --current --namespace=confluent
  ```
* This repo cloned to your workstation:

To complete this scenario, you'll follow these steps:

1. Set the current tutorial directory.

2. Deploy Confluent For Kubernetes.

3. Deploy configuration secrets.

4. Deploy Confluent Platform.

5. Deploy the Producer application.

6. Validate.

7. Tear down Confluent Platform.

# Set the current tutorial directory

Set the tutorial directory for this tutorial under the directory you downloaded
the tutorial files:
    ```
    export CFK_HOME=<path to the cloned repository>
    ```

# Deploy Confluent for Kubernetes

1. Set up the Helm Chart:

   ```
   helm repo add confluentinc https://packages.confluent.io/helm
   ```

2. Install Confluent For Kubernetes using Helm:

    ```
    helm upgrade --install operator confluentinc/confluent-for-kubernetes -n confluent --set kRaftEnabled=true
    ```
    #### NOTE:  
    Make sure to deploy CFK with the `–-set kRaftEnabled=true` flag in the helm upgrade command so that CFK can create the required ClusterRole and ClusterRolebinding for KRaft controllers. Please check the following documentation link for more information: 

    - [Deploy CFK with KRaft](https://docs.confluent.io/operator/current/co-deploy-cfk.html#deploy-co-with-kraft)
                
3. Check that the Confluent For Kubernetes pod comes up and is running:

    ```
    kubectl get pods -n confluent
    ```

# Deploy configuration secrets

You'll use Kubernetes secrets to provide credential configurations.

With Kubernetes secrets, credential management (defining, configuring, updating)
can be done outside of the Confluent For Kubernetes. You define the configuration
secret, and then tell Confluent For Kubernetes where to find the configuration.

### Provide authentication credentials

Create a Kubernetes secret object for Zookeeper, Kafka, and Control Center. This
secret object contains file based properties. These files are in the format that
each respective Confluent component requires for authentication credentials.

  ```
  kubectl create secret generic credential \
  --from-file=plain-users.json=$CFK_HOME/creds-kafka-sasl-users.json \
  --from-file=digest.txt=$CFK_HOME/creds-kafka-kraft-credentials.txt \
  --from-file=plain.txt=$CFK_HOME/creds-client-kafka-sasl-user.txt \
  --from-file=basic.txt=$CFK_HOME/creds-control-center-users.txt
  ```

In this tutorial, we use one credential for authenticating all client and server
communication to Kafka brokers. In production scenarios, you'll want to specify
different credentials for each of them.

# Review Confluent Platform configurations

You install Confluent Platform components as custom resources (CRs). 

The Confluent Platform components are configured in one file
``$CFK_HOME/confluent-platform.yaml``

Let's take a look at how these components are configured.

* Configure SASL PLAIN authentication for Kafka, with a pointer to the externally managed secrets object for credentials:
    ```
    spec:
      listeners:
        internal:
          authentication:
            type: plain
            jaasConfig:
              secretRef: credential
    ```

* Configure SASL PLAIN authentication to Kafka for other components, using a pointer to the externally managed secrets object for credentials:
 
    ```
    spec:
      dependencies:
        kafka:
          bootstrapEndpoint: kafka.confluent.svc.cluster.local:9071
          authentication:
            type: plain
            jaasConfig:
              secretRef: credential
    ```
  
# Deploy Confluent Platform

1. Deploy Confluent Platform with the above configuration:

    ```
    kubectl apply -f $CFK_HOME/confluent-platform.yaml
    ```

2. Check that all Confluent Platform resources are deployed:

    ```
    kubectl get confluent
    kubectl get pods
    ```

3. Get the status of any component. For example, to check Control Center:

    ```
    kubectl describe controlcenter
    ```

# Provide client configurations

You'll need to provide the client configurations to use. This can be provided as
a Kubernetes secret that client applications can use.

1. Get the status of Kafka:

    ```
    kubectl describe kafka
    ```
  
2. In the output of the previous command, validate the internal client config:

    ```
    Listeners:
    Internal:
        Authentication Type: plain
        Client: bootstrap.servers=kafka.confluent.svc.cluster.local:9071
    ```

3. Create the ``kafka.properties`` file in $CFK_HOME. Add the above endpoint and the credentials as follows:
  
    ```
    bootstrap.servers=kafka.confluent.svc.cluster.local:9071
    sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username=kafka_client password=kafka_client-secret;
    sasl.mechanism=PLAIN
    security.protocol=SASL_PLAINTEXT
    ```

4. Create a configuration secret for client applications to use:

    ```
    kubectl create secret generic kafka-client-config \
       --from-file=$CFK_HOME/kafka.properties
    ```
  
# Validate

### Deploy the producer application

Now that we've got the infrastructure set up, let's deploy the producer client
app.

The producer app is packaged and deployed as a pod on Kubernetes. The required
topic is defined as a KafkaTopic custom resource in
``$CFK_HOME/secure-producer-app-data.yaml``.

This app takes the above client configuration as a Kubernetes secret. The secret
is mounted to the app pod file system, and the client application reads the
configuration as a file.

  ```
  kubectl apply -f $CFK_HOME/producer-app-data.yaml
  ```

### Validate in Control Center

Use Control Center to monitor the Confluent Platform, and see the created topic
and data.

1. Set up port forwarding to Control Center web UI from local machine:

    ```
    kubectl port-forward controlcenter-0 9021:9021
    ```

2. Browse to Control Center and log in as the ``admin`` user with the ``Developer1`` password:

    ```
    http://localhost:9021
    ```

3. Check that the ``elastic-0`` topic was created and that messages are being produced to the topic.

# Tear down

  ```
  kubectl delete -f $CFK_HOME/producer-app-data.yaml
  kubectl delete -f $CFK_HOME/confluent-platform.yaml
  kubectl delete secret kafka-client-config
  kubectl delete secret credential
  helm delete operator
  ```