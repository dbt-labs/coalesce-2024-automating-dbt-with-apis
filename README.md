Repository for the Coalesce 2024 training: "Automating dbt Cloud Project Administration & Discovery with APIs"

[Slides](https://docs.google.com/presentation/d/1sEwKVngqw8pvfXW2XmIvCp8hZDrkUvOG511e-g908w0/edit#slide=id.g283b909a8d7_0_0) or bit.ly/c24-apis 

## Code snippets

```
curl -L https://rickandmortyapi.com/api/character 
```

```
curl -O -L https://rickandmortyapi.com/api/character
```

```
import requests

resp = requests.get("https://rickandmortyapi.com/api/character")

print("--STATUS CODE--")
print(resp.status_code)

print("\n\n--TEXT RESPONSE--")
print(resp.text)

print("\n\n--EXTRACT COUNT DATA--")
parsed_response = resp.json()
print(parsed_response["info"]["count"])
```

```
curl https://cloud.getdbt.com/api/v3/accounts/<accountid>/users/ --header 'Authorization: Token <yourtoken>
```

```
import pprint
import requests

my_headers = {"Authorization": "Token <yourtoken>"}

resp = requests.get("https://cloud.getdbt.com/api/v3/accounts/<accountid>/users/", headers = my_headers)


users_data = resp.json()


pprint.pp(users_data)
```

```
import pprint
import requests

my_headers = {"Authorization": "Token TOK"}

my_data = {"cause": "Called from <yourname>"}

resp = requests.post("https://th031.us1.dbt.com/api/v2/accounts/70403103954898/jobs/70403104202956/run", headers = my_headers, json=my_data)

job_trigger_data = resp.json()

pprint.pp(job_trigger_data)
```

```
import pprint
import requests

my_headers = {"Authorization": "Token TOK"}

resp = requests.get("https://th031.us1.dbt.com/api/v2/accounts/70403103954898/runs/", headers = my_headers)

job_trigger_data = resp.json()

pprint.pp(job_trigger_data)
```

### Terraform

provider.tf
```hcl
variable "dbt_account_id" {
  type = number
}

variable "dbt_token" {
  type = string
}

variable "dbt_host_url" {
  type = string
}

terraform {
  required_providers {
    dbtcloud = {
      source = "dbt-labs/dbtcloud"
      version = "~> 0.3"
    }
  }
}

provider "dbtcloud" {
  account_id = var.dbt_account_id
  token = var.dbt_token
  host_url = var.dbt_host_url
}
```

terraform.tfvars
```
dbt_account_id = <your-account-id>
dbt_token      = "<your-token>"
dbt_host_url   = "https://cloud.getdbt.com/api"
```

main.tf
```hcl
// create a project
resource "dbtcloud_project" "my_project" {
  name = "My Coalesce workshop project"
}


// create a global connection
resource "dbtcloud_global_connection" "my_connection" {
  name       = "My Snowflake warehouse"
  snowflake = {
    account    = "my-snowflake-account"
    database   = "MY_DATABASE"
    role       = "MY_ROLE"
    warehouse  = "MY_WAREHOUSE"
  }
}

// link a repository to the dbt Cloud project
// this example adds a github repo for which we know the installation_id but the resource docs have other examples
resource "dbtcloud_repository" "my_repository" {
  project_id             = dbtcloud_project.my_project.id
  remote_url             = "git@github.com:dbt-labs/jaffle-shop-classic.git"
  git_clone_strategy     = "deploy_key"
}

resource "dbtcloud_project_repository" "my_project_repository" {
  project_id    = dbtcloud_project.my_project.id
  repository_id = dbtcloud_repository.my_repository.repository_id
}


// create 2 environments, one for Dev and one for Prod
// here both are linked to the same Data Warehouse connection
// for Prod, we need to create a credential as well
resource "dbtcloud_environment" "my_dev" {
  dbt_version     = "versionless"
  name            = "Dev"
  project_id      = dbtcloud_project.my_project.id
  type            = "development"
  connection_id   = dbtcloud_global_connection.my_connection.id
}

resource "dbtcloud_environment" "my_prod" {
  dbt_version     = "versionless"
  name            = "Prod"
  project_id      = dbtcloud_project.my_project.id
  type            = "deployment"
  deployment_type = "production"
  credential_id   = dbtcloud_snowflake_credential.prod_credential.credential_id
  connection_id   = dbtcloud_global_connection.my_connection.id
}

// we use user/password but there are other options on the resource docs
resource "dbtcloud_snowflake_credential" "prod_credential" {
  project_id  = dbtcloud_project.my_project.id
  auth_type   = "password"
  num_threads = 16
  schema      = "analytics"
  user        = "my_snowflake_user"
  // note, this is a simple example to get Terraform and dbt Cloud working, but do not store passwords in the config for a real productive use case
  // there are different strategies available to protect sensitive input: https://developer.hashicorp.com/terraform/tutorials/configuration-language/sensitive-variables
  password    = "my_snowflake_password"
}
```

append to main.tf
```hcl
resource "dbtcloud_job" "daily_job" {
  environment_id = dbtcloud_environment.prod_environment.environment_id
  execute_steps = [
    "dbt build"
  ]
  generate_docs        = true
  name                 = "Daily job"
  num_threads          = 16
  project_id           = dbtcloud_project.dbt_project.id
  run_generate_sources = true
  target_name          = "default"
  triggers = {
    "github_webhook" : false
    "git_provider_webhook" : false
    "schedule" : false
    "on_merge" : false
  }
  # this is the default that gets set up when modifying jobs in the UI
  schedule_days  = [0, 1, 2, 3, 4, 5, 6]
  schedule_type  = "days_of_week"
  schedule_hours = [0]
}
```

### GraphQL

```
query ($environmentId: BigInt!, $first: Int!) {
 environment(id: $environmentId) {
   applied {
     models(first: $first) {
       edges {
         node {
           uniqueId
           compiledCode
           database
           schema
           alias
           materializedType
           executionInfo {
             executeCompletedAt
             lastJobDefinitionId
             lastRunGeneratedAt
             lastRunId
             lastRunStatus
             lastRunError
             lastSuccessJobDefinitionId
             runGeneratedAt
             lastSuccessRunId
           }}}}}}}

```

```
{
 "environmentId": 70403103966025,
 "first": 100
}
```

```
import requests
import pprint

headers = {'Authorization': 'Bearer TOK'}

graphql_query = """
query ($environmentId: BigInt!, $first: Int) {
    environment(id: $environmentId) {
      applied {
        tests(first: $first) {
          edges {
            node {
              uniqueId
              filePath
              executionInfo {
                executionTime
              }
            }
          }
        }
      }
    }
  }
"""

json_data = {
    'query': graphql_query,
    'variables': {
        'environmentId': 70403103966025,
        'first': 100,
    },
}

response = requests.post('https://th031.metadata.us1.dbt.com/graphql', headers=headers, json=json_data)
pprint.pp(response.json())
```

Bonus
```
import requests
import pprint

headers = {'Authorization': 'Bearer TOK'}

graphql_query = ...
...
...


response = requests.post('https://th031.metadata.us1.dbt.com/graphql', headers=headers, json=json_data)

tests_metadata = response.json()["data"]["environment"]["applied"]["tests"]["edges"]

sorted_data = sorted(tests_metadata, key=lambda x: x['node']['executionInfo']['executionTime'], reverse=True)

for item in sorted_data[:5]:
    print(f"{item["node"]["uniqueId"]} -- {item["node"]["executionInfo"]["executionTime"]} seconds")
```
