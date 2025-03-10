# Testing of the `syndeo-template` package

## Testing the Kafka-to-BigQuery solution

The end-to-end test for the Kafka-to-BigQuery solution is implemented in `KafkaToBigQueryIT#testKickOffKafkaToBigQuerySyndeoPipeline`.
The test performs the following set of actions:

1. Start a pipeline that generates random data and writes it to a Kafka cluster as Avro-encoded records (data generator)
   1. **Note**: This pipeline generates a fixed number of records.
2. Start a pipeline from the Syndeo template that will consume data from the Kafka cluster (pipeline #1)
3. Let the pipeline run for a short time
4. Drain pipeline #1
5. Start a new pipeline with the Syndeo template with the same configuration as #1 (this new pipeline is pipeline #2)
6. Query BigQuery until it contains at least the expected number of records.
7. Terminate the second Syndeo pipeline
8. (implicit) The data generator will terminate on its own after generating the fixed number of elements.

After performing those actions, the test then verifies that only the expected number of records were written to BigQuery
(no data was duplicated.)

The end-to-end test for the Kafka-to-BigQuery solution assumes various pieces of infrastructure exist, and relies on
them to run. The infrastructure that the test expects is:

- A locally set-up `DT_IT_ACCESS_TOKEN` variable that allows the testing framework to authenticate to GCP.
    - If this access token is not provided, GCP application default credentials are used.
- A kafka cluster with a bootstrap server on `kafka-pabloem-sept-2022-test-m:9092` - note that the IP address is
    resolved in GCP
- A BigQuery dataset called `syndeo_test` under the project that the template will run (normally `cloud-teleport-testing`).
- The template spec staged in `gs://cloud-teleport-testing-df-staging/syndeo-template.json`
- The template container staged in `gcr.io/apache-beam-testing/syndeo-template:latest`
- **Note**: The test has not been run outside of the Template testing project (`cloud-teleport-testing`).

### Create a Kafka cluster
TODO(pabloem): Fill this up.
### Push the template to its appropriate location

To build and push the Syndeo template artifacts, you must run the following commands:

Build the uber JAR containing the Syndeo template and its dependencies. This will output the uber JAR
as `syndeo-template/target/syndeo-template-1.0-SNAPSHOT.jar`.

```shell
mvn -B clean package -DskipTests -am -f  unified-templates.xml -pl syndeo-template/pom.xml
```

Using this JAR, we can now build the template and push it to GCP. Note that the `syndeo-template/metadata.json` file
contains the configuration parameters for the template, and they can be reviewed and modified there.

```shell
gcloud dataflow flex-template build $GCS_BUCKET/syndeo-template.json  \
    --metadata-file syndeo-template/metadata.json \
    --sdk-language "JAVA" \
    --flex-template-base-image JAVA11 \
    --image-gcr-path=gcr.io/$GCP_PROJECT/syndeo-template:latest \
    --jar "syndeo-template/target/syndeo-template-1.0-SNAPSHOT.jar" \
    --env FLEX_TEMPLATE_JAVA_MAIN_CLASS="com.google.cloud.syndeo.SyndeoTemplate"
```

### Trigger the test

The testing framework relies on the `DT_IT_TOKEN` variable to authenticate, so you'll need to set that up first:

```shell
export DT_IT_ACCESS_TOKEN=$(gcloud auth application-default print-access-token)
```

Once all infrastructure is set up, the test can be triggered.

```shell
mvn clean package test -f unified-templates.xml -pl syndeo-template/pom.xml  \
    -Dtest="KafkaToBigQueryIT#testKickOffKafkaToBigQuerySyndeoPipeline" \
    -Dproject="cloud-teleport-testing" \
    -DartifactBucket=${TEMP_LOCATION} \
    -Dregion="us-central1" \
    -DtempLocation=${TEMP_LOCATION}
```




## Resources
- [Running a small Kafka deployment for testing on GCP](https://medium.com/google-cloud/setting-up-a-small-kafka-server-on-google-cloud-platform-for-testing-purposes-9958a47ea8b9)
