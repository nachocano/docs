# Google Cloud Pub/Sub Importer

This importer brings Google Cloud Pub/Sub events into Knative. 
As many Google Cloud Services use Pub/Sub as a transport, 
this importer can be used to bring events from services such as [Cloud Storage](https://cloud.google.com/storage/docs/pubsub-notifications), 
[IoT Core](https://cloud.google.com/iot/docs/how-tos/devices), [Cloud Scheduler](https://cloud.google.com/scheduler/docs/creating#), etc.

## Before you begin

1. Create a
   [Google Cloud project](https://cloud.google.com/resource-manager/docs/creating-managing-projects)
   and install the `gcloud` CLI and run `gcloud auth login`. This example will
   use a mix of `gcloud` and `kubectl` commands. The rest of the example assumes
   that you've set the `$PROJECT_ID` environment variable to your Google Cloud
   project ID, and also set your project ID as default using
   `gcloud config set project $PROJECT_ID`.
1. [Install Knative Eventing](#) and verify the health of the Knative Eventing control plane.
1. Install the Google Cloud Pub/Sub importer. Note that we are using `v.0.6.0` in the following command. 

   ```shell
   kubectl apply --filename https://github.com/knative/eventing-sources/releases/download/v0.6.0/gcppubsub.yaml
   ```

1. Enable the Google Cloud Pub/Sub API on your project:

   ```shell
   gcloud services enable pubsub.googleapis.com
   ```

1. Create a
   [GCP Service Account](https://console.cloud.google.com/iam-admin/serviceaccounts/project).
   This sample creates one service account for both registration and receiving
   messages, but you can also create a separate service account for receiving
   messages if you want additional privilege separation.

   1. Create a new service account named `knative-source` with the following
      command:
      ```shell
      gcloud iam service-accounts create knative-source
      ```
   1. Give that Service Account the `Pub/Sub Editor` role on your GCP project:
      ```shell
      gcloud projects add-iam-policy-binding $PROJECT_ID \
        --member=serviceAccount:knative-source@$PROJECT_ID.iam.gserviceaccount.com \
        --role roles/pubsub.editor
      ```
   1. Download a new JSON private key for that Service Account.
      ```shell
      gcloud iam service-accounts keys create knative-source.json \
        --iam-account=knative-source@$PROJECT_ID.iam.gserviceaccount.com
      ```
   1. Create two secrets on the kubernetes cluster with the downloaded key:

      ```shell
      # Note that the first secret may already have been created when installing
      # Knative Eventing. The following command will overwrite it. If you don't
      # want to overwrite it, then skip this command.
      kubectl --namespace knative-sources create secret generic gcppubsub-source-key --from-file=key.json=knative-source.json --dry-run --output yaml | kubectl apply --filename -

      # The second secret should not already exist, so just try to create it.
      kubectl --namespace default create secret generic google-cloud-key --from-file=key.json=knative-source.json
      ```

      `gcppubsub-source-key` and `key.json` are pre-configured values in the
      `gcppubsub-controller-manager` StatefulSet that manages your Google Cloud Pub/Sub importers.

      `google-cloud-key` and `key.json` are pre-configured values in the examples 
      we will use throughout this tutorial.

## Description

This sample will create the following topology: 

TODO Image.

 * A Google Cloud Pub/Sub Importer called `pubsub-test` that subscribes to the `testing` topic in your 
 Google Cloud project.
 * A Broker named default, where the `pubsub-test` Importer sends the events to.
 * A Trigger that listens to Google Cloud Pub/Sub events from the `default` Broker.
 * A Kubernetes Service pointing to a Deployment that acts as the event consumer. 

## Step-by-step

1. Label the `default` namespace so that the `default` Broker is created. These instructions assume the
   namespace `default`, feel free to change to any other namespace you would
   like to use instead:

   ```shell
   kubectl label namespace default knative-eventing-injection=enabled
   ```
   
1. Verify the Broker is ready.

   ```shell
   kubectl get broker namespace default
   ```


1. Create a Google Cloud Pub/Sub topic called `testing`:

   ```shell
   gcloud pubsub topics create testing
   ```

1. Replace `MY_GCP_PROJECT` with your Google Cloud Project in the following snippet and 
install the importer. You can save the snippet below into its own yaml file and then do 
`kubectl apply -f <filename>`.

    ```yaml
    apiVersion: sources.eventing.knative.dev/v1alpha1
    kind: GcpPubSubSource
    metadata:
      name: pubsub-test
      namespace: default
    spec:
      gcpCredsSecret:
        name: google-cloud-key
        key: key.json
      googleCloudProject: MY_GCP_PROJECT  # Replace this
      topic: testing
      sink:
        apiVersion: eventing.knative.dev/v1alpha1
        kind: Broker
        name: default
    ```

1. Verify the importer was installed and is ready. You have to check in the `Status` conditions for the 
`Ready` condition.

    ```shell
    kubectl describe gcppubsubsources pubsub-test
    ```
    
1. Discover the events you can now consume by querying the registry.

   ```shell
   kubectl get eventtypes -n default
   ```
   
   You should see an output similar to the following one: 

    ```
    NAME                                   TYPE                                SOURCE                                              SCHEMA        BROKER      DESCRIPTION     READY     REASON
    google.pubsub.topic.publish-hrxhh      google.pubsub.topic.publish         //pubsub.googleapis.com/knative/topics/testing                    default                     True      
    ```

1. Create the Deployment and Service that will act as the event consumer. You can save the snippets below into 
their own yaml files and then apply them with `kubectl apply -f <filename>`. 

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: event-display
    spec:
      replicas: 1
      selector:
        matchLabels: &labels
          app: event-display
      template:
        metadata:
          labels: *labels
        spec:
          containers:
            - name: event-display
              image: gcr.io/knative-releases/github.com/knative/eventing-sources/cmd/event_display@sha256:37ace92b63fc516ad4c8331b6b3b2d84e4ab2d8ba898e387c0b6f68f0e3081c4
    ```
    
    ```yaml
    kind: Service
    apiVersion: v1
    metadata:
      name: event-display
    spec:
      selector:
        app: event-display
      ports:
      - protocol: TCP
        port: 80
        targetPort: 8080
    ```    

1. Verify the deployment was created and is ready:

    ```shell
    kubectl get deployments -n default
    ```

    In particular, we want the number in the DESIRED column to match the number in the AVAILABLE column. This may take 
    few minutes.    
    
1. Verify the service was created:

    ```shell
    kubectl get services -n default
    ```    

1. Create a Trigger to subscribe to the Google Cloud Pub/Sub events from the `testing` topic on the `default` Broker:

    ```yaml
    apiVersion: eventing.knative.dev/v1alpha1
    kind: Trigger
    metadata:
      name: my-trigger
      namespace: default
    spec:
      broker: default
      filter:
        sourceAndType:
          type: google.pubsub.topic.publish
          source: //pubsub.googleapis.com/knative/topics/testing
      subscriber:
        ref:
         apiVersion: v1
         kind: Service
         name: event-display
    ```
    
1. Verify the Trigger was created and is ready:
    
    ```shell
    kubectl get triggers -n default
    ```
    
1. Publish a message to your Google Cloud Pub/Sub Topic:

    ```shell
    gcloud pubsub topics publish testing --message="Hello world"
    ```

1. Verify that the published message was sent into your Knative cluster by looking at the logs of the `event-display` pod.

    ```shell
    kubectl -n kn-eventing-registry-demo logs -l app=event-display
    ```
    You should see log lines similar to:

    ```
    ☁️   cloudevents.Event
    Validation: valid
    Context Attributes,
      specversion: 0.2
      type: google.pubsub.topic.publish
      source: //pubsub.googleapis.com/knative/topics/testing
      id: c9ba9ac5-3210-4813-841f-da2b717efsa7
      time: 2019-05-17T19:00:00.000655717Z
      contenttype: application/json
    Extensions,
      knativehistory: default-broker-d4qjc-channel-2gthx.default.svc.cluster.local
    Data,
        {
          "ID": "284375451531353",
          "Data": "SGVsbG8sIHdvcmxk",
          "Attributes": null,
          "PublishTime": "2019-05-17T19:00:00.000655717Z"
        }
    ```

    The log message is a dump of the CloudEvent sent by the Google Cloud Pub/Sub Importer. 
    In particular, if you [base-64 decode](https://www.base64decode.org/) the `Data` field, you should
    see the sent message:

    ```shell
    echo "SGVsbG8sIHdvcmxk" | base64 --decode
    ```

    Results in: "Hello world".
    
## Next Steps

1. Experiment with other Event Importers in your Knative cluster. <Link to GitHubSource, KafkaSource, etc.>
1. ...         