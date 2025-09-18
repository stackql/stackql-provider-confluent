# `confluent` provider for [`stackql`](https://github.com/stackql/stackql)

This repository is used to generate and document the `confluent` provider for StackQL, allowing you to query and manage Confluent Cloud resources using SQL-like syntax. The provider is built using the `@stackql/provider-utils` package, which provides tools for converting OpenAPI specifications into StackQL-compatible provider schemas.

## Prerequisites

To use the Confluent provider with StackQL, you'll need:

1. A Confluent Cloud account with appropriate API credentials
2. Confluent Cloud API key and secret with sufficient permissions for the resources you want to access
3. StackQL CLI installed on your system (see [StackQL](https://github.com/stackql/stackql))

## 1. Download the Open API Specification

First, download the Confluent Cloud API OpenAPI specification:

```bash
mkdir -p provider-dev/downloaded
curl -L https://docs.confluent.io/cloud/current/api.html#section/OpenAPI-Specification/Consumer-OpenAPI-specification \
  -o provider-dev/downloaded/confluent-openapi.yaml

# Convert YAML to JSON if needed
python3 provider-dev/scripts/yaml_to_json.py \
  --input provider-dev/downloaded/confluent-openapi.yaml \
  --output provider-dev/downloaded/openapi.json
```

## 2. Split into Service Specs

Next, split the monolithic OpenAPI specification into service-specific files:

```bash
rm -rf provider-dev/source/*
npm run split -- \
  --provider-name confluent \
  --api-doc provider-dev/downloaded/openapi.json \
  --svc-discriminator tag \
  --output-dir provider-dev/source \
  --overwrite \
  --svc-name-overrides "$(cat <<EOF
{
  "kafka-clusters": "kafka",
  "ksql-clusters": "ksql",
  "schema-registry-clusters": "schema_registry",
  "connect-clusters": "connect",
  "environments": "environments",
  "service-accounts": "service_accounts",
  "api-keys": "api_keys",
  "topics": "topics",
  "acls": "acls",
  "connectors": "connectors",
  "cluster-links": "cluster_links",
  "networking": "networking",
  "billing": "billing"
}
EOF
)"
```

## 3. Generate Mappings

Generate the mapping configuration that connects OpenAPI operations to StackQL resources:

```bash
npm run generate-mappings -- \
  --provider-name confluent \
  --input-dir provider-dev/source \
  --output-dir provider-dev/config
```

Update the resultant `provider-dev/config/all_services.csv` to add the `stackql_resource_name`, `stackql_method_name`, `stackql_verb` values for each operation.

## 4. Generate Provider

This step transforms the split OpenAPI service specs into a fully-functional StackQL provider by applying the resource and method mappings defined in your CSV file.

```bash
rm -rf provider-dev/openapi/*
npm run generate-provider -- \
  --provider-name confluent \
  --input-dir provider-dev/source \
  --output-dir provider-dev/openapi/src/confluent \
  --config-path provider-dev/config/all_services.csv \
  --servers '[{"url": "https://api.confluent.cloud"}]' \
  --provider-config '{"auth": {"type": "basic", "credentials": {"usernameEnvVar": "CONFLUENT_API_KEY", "passwordEnvVar": "CONFLUENT_API_SECRET"}}}' \
  --overwrite
```

## 5. Test Provider

### Starting the StackQL Server

Before running tests, start a StackQL server with your provider:

```bash
PROVIDER_REGISTRY_ROOT_DIR="$(pwd)/provider-dev/openapi"
npm run start-server -- --provider confluent --registry $PROVIDER_REGISTRY_ROOT_DIR
```

### Test Meta Routes

Test all metadata routes (services, resources, methods) in the provider:

```bash
npm run test-meta-routes -- confluent --verbose
```

When you're done testing, stop the StackQL server:

```bash
npm run stop-server
```

Use this command to view the server status:

```bash
npm run server-status
```

### Run test queries

Run some test queries against the provider using the `stackql shell`:

```bash
PROVIDER_REGISTRY_ROOT_DIR="$(pwd)/provider-dev/openapi"
REG_STR='{"url": "file://'${PROVIDER_REGISTRY_ROOT_DIR}'", "localDocRoot": "'${PROVIDER_REGISTRY_ROOT_DIR}'", "verifyConfig": {"nopVerify": true}}'
./stackql shell --registry="${REG_STR}"
```

Example queries to try:

```sql
-- List all environments
SELECT 
id,
display_name,
creation_method,
lifecycle
FROM confluent.environments.environments;

-- List Kafka clusters
SELECT 
id,
display_name,
availability,
bootstrap_endpoint,
cloud,
region,
provider,
status,
environment.id
FROM confluent.kafka.clusters;

-- List topics in a Kafka cluster
SELECT 
name,
partitions_count,
replication_factor,
configs,
is_internal
FROM confluent.topics.topics
WHERE cluster_id = 'lkc-abcdef';

-- Get Schema Registry clusters
SELECT 
id,
display_name,
package,
endpoint,
status,
provider,
region,
environment.id
FROM confluent.schema_registry.clusters;

-- List service accounts
SELECT 
id,
display_name,
description,
cloud,
status
FROM confluent.service_accounts.service_accounts;

-- List API keys
SELECT 
id,
display_name,
owner,
logical_clusters,
description,
created,
modified,
spec
FROM confluent.api_keys.api_keys;

-- List network resources
SELECT 
id,
display_name,
gcp,
aws,
azure,
dns_domain,
status
FROM confluent.networking.networks;

-- List connectors
SELECT 
id,
name,
connector_class,
config,
status,
tasks
FROM confluent.connect.connectors
WHERE cluster_id = 'lkc-abcdef';
```

## 6. Publish the provider

To publish the provider push the `confluent` dir to `providers/src` in a feature branch of the [`stackql-provider-registry`](https://github.com/stackql/stackql-provider-registry). Follow the [registry release flow](https://github.com/stackql/stackql-provider-registry/blob/dev/docs/build-and-deployment.md).  

Launch the StackQL shell:

```bash
export DEV_REG="{ \"url\": \"https://registry-dev.stackql.app/providers\" }"
./stackql --registry="${DEV_REG}" shell
```

Pull the latest dev `confluent` provider:

```sql
registry pull confluent;
```

Run some test queries to verify the provider works as expected.

## 7. Generate web docs

Provider doc microsites are built using Docusaurus and published using GitHub Pages.  

a. Update `headerContent1.txt` and `headerContent2.txt` accordingly in `provider-dev/docgen/provider-data/`  

b. Update the following in `website/docusaurus.config.js`:  

```js
// Provider configuration - change these for different providers
const providerName = "confluent";
const providerTitle = "Confluent Provider";
```

c. Then generate docs using...

```bash
npm run generate-docs -- \
  --provider-name confluent \
  --provider-dir ./provider-dev/openapi/src/confluent/v00.00.00000 \
  --output-dir ./website \
  --provider-data-dir ./provider-dev/docgen/provider-data
```  

## 8. Test web docs locally

```bash
cd website
# test build
yarn build

# run local dev server
yarn start
```

## 9. Publish web docs to GitHub Pages

Under __Pages__ in the repository, in the __Build and deployment__ section select __GitHub Actions__ as the __Source__. In Netlify DNS create the following records:

| Source Domain | Record Type  | Target |
|---------------|--------------|--------|
| confluent-provider.stackql.io | CNAME | stackql.github.io. |

## License

MIT

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.