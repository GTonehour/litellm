import Image from '@theme/IdealImage';
import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Proxy Config.yaml
Set model list, `api_base`, `api_key`, `temperature` & proxy server settings (`master-key`) on the config.yaml. 

| Param Name           | Description                                                   |
|----------------------|---------------------------------------------------------------|
| `model_list`         | List of supported models on the server, with model-specific configs |
| `router_settings`   | litellm Router settings, example `routing_strategy="least-busy"` [**see all**](#router-settings)|
| `litellm_settings`   | litellm Module settings, example `litellm.drop_params=True`, `litellm.set_verbose=True`, `litellm.api_base`, `litellm.cache` [**see all**](#all-settings)|
| `general_settings`   | Server settings, example setting `master_key: sk-my_special_key` |
| `environment_variables`   | Environment Variables example, `REDIS_HOST`, `REDIS_PORT` |

**Complete List:** Check the Swagger UI docs on `<your-proxy-url>/#/config.yaml` (e.g. http://0.0.0.0:4000/#/config.yaml), for everything you can pass in the config.yaml.


## Quick Start 

Set a model alias for your deployments. 

In the `config.yaml` the model_name parameter is the user-facing name to use for your deployment. 

In the config below:
- `model_name`: the name to pass TO litellm from the external client  
- `litellm_params.model`: the model string passed to the litellm.completion() function

E.g.: 
- `model=vllm-models` will route to `openai/facebook/opt-125m`. 
- `model=gpt-3.5-turbo` will load balance between `azure/gpt-turbo-small-eu` and `azure/gpt-turbo-small-ca`

```yaml
model_list:
  - model_name: gpt-3.5-turbo ### RECEIVED MODEL NAME ###
    litellm_params: # all params accepted by litellm.completion() - https://docs.litellm.ai/docs/completion/input
      model: azure/gpt-turbo-small-eu ### MODEL NAME sent to `litellm.completion()` ###
      api_base: https://my-endpoint-europe-berri-992.openai.azure.com/
      api_key: "os.environ/AZURE_API_KEY_EU" # does os.getenv("AZURE_API_KEY_EU")
      rpm: 6      # [OPTIONAL] Rate limit for this deployment: in requests per minute (rpm)
  - model_name: bedrock-claude-v1 
    litellm_params:
      model: bedrock/anthropic.claude-instant-v1
  - model_name: gpt-3.5-turbo
    litellm_params:
      model: azure/gpt-turbo-small-ca
      api_base: https://my-endpoint-canada-berri992.openai.azure.com/
      api_key: "os.environ/AZURE_API_KEY_CA"
      rpm: 6
  - model_name: anthropic-claude
    litellm_params: 
      model: bedrock/anthropic.claude-instant-v1
      ### [OPTIONAL] SET AWS REGION ###
      aws_region_name: us-east-1
  - model_name: vllm-models
    litellm_params:
      model: openai/facebook/opt-125m # the `openai/` prefix tells litellm it's openai compatible
      api_base: http://0.0.0.0:4000/v1
      api_key: none
      rpm: 1440
    model_info: 
      version: 2
  
  # Use this if you want to make requests to `claude-3-haiku-20240307`,`claude-3-opus-20240229`,`claude-2.1` without defining them on the config.yaml
  # Default models
  # Works for ALL Providers and needs the default provider credentials in .env
  - model_name: "*" 
    litellm_params:
      model: "*"

litellm_settings: # module level litellm settings - https://github.com/BerriAI/litellm/blob/main/litellm/__init__.py
  drop_params: True
  success_callback: ["langfuse"] # OPTIONAL - if you want to start sending LLM Logs to Langfuse. Make sure to set `LANGFUSE_PUBLIC_KEY` and `LANGFUSE_SECRET_KEY` in your env

general_settings: 
  master_key: sk-1234 # [OPTIONAL] Only use this if you to require all calls to contain this key (Authorization: Bearer sk-1234)
  alerting: ["slack"] # [OPTIONAL] If you want Slack Alerts for Hanging LLM requests, Slow llm responses, Budget Alerts. Make sure to set `SLACK_WEBHOOK_URL` in your env
```
:::info

For more provider-specific info, [go here](../providers/)

:::

#### Step 2: Start Proxy with config

```shell
$ litellm --config /path/to/config.yaml
```

:::tip

Run with `--detailed_debug` if you need detailed debug logs 

```shell
$ litellm --config /path/to/config.yaml --detailed_debug
```

:::

#### Step 3: Test it

Sends request to model where `model_name=gpt-3.5-turbo` on config.yaml. 

If multiple with `model_name=gpt-3.5-turbo` does [Load Balancing](https://docs.litellm.ai/docs/proxy/load_balancing)

**[Langchain, OpenAI SDK Usage Examples](../proxy/user_keys#request-format)**

```shell
curl --location 'http://0.0.0.0:4000/chat/completions' \
--header 'Content-Type: application/json' \
--data ' {
      "model": "gpt-3.5-turbo",
      "messages": [
        {
          "role": "user",
          "content": "what llm are you"
        }
      ],
    }
'
```

## LLM configs `model_list`

### Model-specific params (API Base, Keys, Temperature, Max Tokens, Organization, Headers etc.)
You can use the config to save model-specific information like api_base, api_key, temperature, max_tokens, etc. 

[**All input params**](https://docs.litellm.ai/docs/completion/input#input-params-1)

**Step 1**: Create a `config.yaml` file
```yaml
model_list:
  - model_name: gpt-4-team1
    litellm_params: # params for litellm.completion() - https://docs.litellm.ai/docs/completion/input#input---request-body
      model: azure/chatgpt-v-2
      api_base: https://openai-gpt-4-test-v-1.openai.azure.com/
      api_version: "2023-05-15"
      azure_ad_token: eyJ0eXAiOiJ
      seed: 12
      max_tokens: 20
  - model_name: gpt-4-team2
    litellm_params:
      model: azure/gpt-4
      api_key: sk-123
      api_base: https://openai-gpt-4-test-v-2.openai.azure.com/
      temperature: 0.2
  - model_name: openai-gpt-3.5
    litellm_params:
      model: openai/gpt-3.5-turbo
      extra_headers: {"AI-Resource Group": "ishaan-resource"}
      api_key: sk-123
      organization: org-ikDc4ex8NB
      temperature: 0.2
  - model_name: mistral-7b
    litellm_params:
      model: ollama/mistral
      api_base: your_ollama_api_base
```

**Step 2**: Start server with config

```shell
$ litellm --config /path/to/config.yaml
```

**Expected Logs:**

Look for this line in your console logs to confirm the config.yaml was loaded in correctly.
```
LiteLLM: Proxy initialized with Config, Set models:
```

### Embedding Models - Use Sagemaker, Bedrock, Azure, OpenAI, XInference

See supported Embedding Providers & Models [here](https://docs.litellm.ai/docs/embedding/supported_embedding)


<Tabs>
<TabItem value="bedrock" label="Bedrock Completion/Chat">

```yaml
model_list:
  - model_name: bedrock-cohere
    litellm_params:
      model: "bedrock/cohere.command-text-v14"
      aws_region_name: "us-west-2"
  - model_name: bedrock-cohere
    litellm_params:
      model: "bedrock/cohere.command-text-v14"
      aws_region_name: "us-east-2"
  - model_name: bedrock-cohere
    litellm_params:
      model: "bedrock/cohere.command-text-v14"
      aws_region_name: "us-east-1"

```

</TabItem>

<TabItem value="sagemaker" label="Sagemaker, Bedrock Embeddings">

Here's how to route between GPT-J embedding (sagemaker endpoint), Amazon Titan embedding (Bedrock) and Azure OpenAI embedding on the proxy server: 

```yaml
model_list:
  - model_name: sagemaker-embeddings
    litellm_params: 
      model: "sagemaker/berri-benchmarking-gpt-j-6b-fp16"
  - model_name: amazon-embeddings
    litellm_params:
      model: "bedrock/amazon.titan-embed-text-v1"
  - model_name: azure-embeddings
    litellm_params: 
      model: "azure/azure-embedding-model"
      api_base: "os.environ/AZURE_API_BASE" # os.getenv("AZURE_API_BASE")
      api_key: "os.environ/AZURE_API_KEY" # os.getenv("AZURE_API_KEY")
      api_version: "2023-07-01-preview"

general_settings:
  master_key: sk-1234 # [OPTIONAL] if set all calls to proxy will require either this key or a valid generated token
```

</TabItem>

<TabItem value="Hugging Face emb" label="Hugging Face Embeddings">
LiteLLM Proxy supports all <a href="https://huggingface.co/models?pipeline_tag=feature-extraction">Feature-Extraction Embedding models</a>.

```yaml
model_list:
  - model_name: deployed-codebert-base
    litellm_params: 
      # send request to deployed hugging face inference endpoint
      model: huggingface/microsoft/codebert-base # add huggingface prefix so it routes to hugging face
      api_key: hf_LdS                            # api key for hugging face inference endpoint
      api_base: https://uysneno1wv2wd4lw.us-east-1.aws.endpoints.huggingface.cloud # your hf inference endpoint 
  - model_name: codebert-base
    litellm_params: 
      # no api_base set, sends request to hugging face free inference api https://api-inference.huggingface.co/models/
      model: huggingface/microsoft/codebert-base # add huggingface prefix so it routes to hugging face
      api_key: hf_LdS                            # api key for hugging face                     

```

</TabItem>

<TabItem value="azure" label="Azure OpenAI Embeddings">

```yaml
model_list:
  - model_name: azure-embedding-model # model group
    litellm_params:
      model: azure/azure-embedding-model # model name for litellm.embedding(model=azure/azure-embedding-model) call
      api_base: your-azure-api-base
      api_key: your-api-key
      api_version: 2023-07-01-preview
```

</TabItem>

<TabItem value="openai" label="OpenAI Embeddings">

```yaml
model_list:
- model_name: text-embedding-ada-002 # model group
  litellm_params:
    model: text-embedding-ada-002 # model name for litellm.embedding(model=text-embedding-ada-002) 
    api_key: your-api-key-1
- model_name: text-embedding-ada-002 
  litellm_params:
    model: text-embedding-ada-002
    api_key: your-api-key-2
```

</TabItem>


<TabItem value="xinf" label="XInference">

https://docs.litellm.ai/docs/providers/xinference

**Note add `xinference/` prefix to `litellm_params`: `model` so litellm knows to route to OpenAI**

```yaml
model_list:
- model_name: embedding-model  # model group
  litellm_params:
    model: xinference/bge-base-en   # model name for litellm.embedding(model=xinference/bge-base-en) 
    api_base: http://0.0.0.0:9997/v1
```

</TabItem>

<TabItem value="openai emb" label="OpenAI Compatible Embeddings">

<p>Use this for calling <a href="https://github.com/xorbitsai/inference">/embedding endpoints on OpenAI Compatible Servers</a>.</p>

**Note add `openai/` prefix to `litellm_params`: `model` so litellm knows to route to OpenAI**

```yaml
model_list:
- model_name: text-embedding-ada-002  # model group
  litellm_params:
    model: openai/<your-model-name>   # model name for litellm.embedding(model=text-embedding-ada-002) 
    api_base: <model-api-base>
```

</TabItem>
</Tabs>

#### Start Proxy

```shell
litellm --config config.yaml
```

#### Make Request
Sends Request to `bedrock-cohere`

```shell
curl --location 'http://0.0.0.0:4000/chat/completions' \
  --header 'Content-Type: application/json' \
  --data ' {
  "model": "bedrock-cohere",
  "messages": [
      {
      "role": "user",
      "content": "gm"
      }
  ]
}'
```


### Multiple OpenAI Organizations 

Add all openai models across all OpenAI organizations with just 1 model definition 

```yaml
  - model_name: *
    litellm_params:
      model: openai/*
      api_key: os.environ/OPENAI_API_KEY
      organization:
       - org-1 
       - org-2 
       - org-3
```

LiteLLM will automatically create separate deployments for each org.

Confirm this via 

```bash
curl --location 'http://0.0.0.0:4000/v1/model/info' \
--header 'Authorization: Bearer ${LITELLM_KEY}' \
--data ''
```

 
### Provider specific wildcard routing 
**Proxy all models from a provider**

Use this if you want to **proxy all models from a specific provider without defining them on the config.yaml**

**Step 1** - define provider specific routing on config.yaml
```yaml
model_list:
  # provider specific wildcard routing
  - model_name: "anthropic/*"
    litellm_params:
      model: "anthropic/*"
      api_key: os.environ/ANTHROPIC_API_KEY
  - model_name: "groq/*"
    litellm_params:
      model: "groq/*"
      api_key: os.environ/GROQ_API_KEY
```

Step 2 - Run litellm proxy 

```shell
$ litellm --config /path/to/config.yaml
```

Step 3 Test it 

Test with `anthropic/` - all models with `anthropic/` prefix will get routed to `anthropic/*`
```shell
curl http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-1234" \
  -d '{
    "model": "anthropic/claude-3-sonnet-20240229",
    "messages": [
      {"role": "user", "content": "Hello, Claude!"}
    ]
  }'
```

Test with `groq/` - all models with `groq/` prefix will get routed to `groq/*`
```shell
curl http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-1234" \
  -d '{
    "model": "groq/llama3-8b-8192",
    "messages": [
      {"role": "user", "content": "Hello, Claude!"}
    ]
  }'
```

### Load Balancing 

:::info
For more on this, go to [this page](https://docs.litellm.ai/docs/proxy/load_balancing)
:::

Use this to call multiple instances of the same model and configure things like [routing strategy](https://docs.litellm.ai/docs/routing#advanced).

For optimal performance:
- Set `tpm/rpm` per model deployment. Weighted picks are then based on the established tpm/rpm.
- Select your optimal routing strategy in `router_settings:routing_strategy`.

LiteLLM supports
```python
["simple-shuffle", "least-busy", "usage-based-routing","latency-based-routing"], default="simple-shuffle"`
```

When `tpm/rpm` is set + `routing_strategy==simple-shuffle` litellm will use a weighted pick based on set tpm/rpm. **In our load tests setting tpm/rpm for all deployments + `routing_strategy==simple-shuffle` maximized throughput**
- When using multiple LiteLLM Servers / Kubernetes set redis settings `router_settings:redis_host` etc

```yaml
model_list:
  - model_name: zephyr-beta
    litellm_params:
        model: huggingface/HuggingFaceH4/zephyr-7b-beta
        api_base: http://0.0.0.0:8001
        rpm: 60      # Optional[int]: When rpm/tpm set - litellm uses weighted pick for load balancing. rpm = Rate limit for this deployment: in requests per minute (rpm).
        tpm: 1000   # Optional[int]: tpm = Tokens Per Minute 
  - model_name: zephyr-beta
    litellm_params:
        model: huggingface/HuggingFaceH4/zephyr-7b-beta
        api_base: http://0.0.0.0:8002
        rpm: 600      
  - model_name: zephyr-beta
    litellm_params:
        model: huggingface/HuggingFaceH4/zephyr-7b-beta
        api_base: http://0.0.0.0:8003
        rpm: 60000      
  - model_name: gpt-3.5-turbo
    litellm_params:
        model: gpt-3.5-turbo
        api_key: <my-openai-key>
        rpm: 200      
  - model_name: gpt-3.5-turbo-16k
    litellm_params:
        model: gpt-3.5-turbo-16k
        api_key: <my-openai-key>
        rpm: 100      

litellm_settings:
  num_retries: 3 # retry call 3 times on each model_name (e.g. zephyr-beta)
  request_timeout: 10 # raise Timeout error if call takes longer than 10s. Sets litellm.request_timeout 
  fallbacks: [{"zephyr-beta": ["gpt-3.5-turbo"]}] # fallback to gpt-3.5-turbo if call fails num_retries 
  context_window_fallbacks: [{"zephyr-beta": ["gpt-3.5-turbo-16k"]}, {"gpt-3.5-turbo": ["gpt-3.5-turbo-16k"]}] # fallback to gpt-3.5-turbo-16k if context window error
  allowed_fails: 3 # cooldown model if it fails > 1 call in a minute. 

router_settings: # router_settings are optional
  routing_strategy: simple-shuffle # Literal["simple-shuffle", "least-busy", "usage-based-routing","latency-based-routing"], default="simple-shuffle"
  model_group_alias: {"gpt-4": "gpt-3.5-turbo"} # all requests with `gpt-4` will be routed to models with `gpt-3.5-turbo`
  num_retries: 2
  timeout: 30                                  # 30 seconds
  redis_host: <your redis host>                # set this when using multiple litellm proxy deployments, load balancing state stored in redis
  redis_password: <your redis password>
  redis_port: 1992
```

You can view your cost once you set up [Virtual keys](https://docs.litellm.ai/docs/proxy/virtual_keys) or [custom_callbacks](https://docs.litellm.ai/docs/proxy/logging)


### Load API Keys / config values from Environment 

If you have secrets saved in your environment, and don't want to expose them in the config.yaml, here's how to load model-specific keys from the environment. **This works for ANY value on the config.yaml**

```yaml
os.environ/<YOUR-ENV-VAR> # runs os.getenv("YOUR-ENV-VAR")
```

```yaml 
model_list:
  - model_name: gpt-4-team1
    litellm_params: # params for litellm.completion() - https://docs.litellm.ai/docs/completion/input#input---request-body
      model: azure/chatgpt-v-2
      api_base: https://openai-gpt-4-test-v-1.openai.azure.com/
      api_version: "2023-05-15"
      api_key: os.environ/AZURE_NORTH_AMERICA_API_KEY # 👈 KEY CHANGE
```

[**See Code**](https://github.com/BerriAI/litellm/blob/c12d6c3fe80e1b5e704d9846b246c059defadce7/litellm/utils.py#L2366)

s/o to [@David Manouchehri](https://www.linkedin.com/in/davidmanouchehri/) for helping with this. 

### Load API Keys from Secret Managers (Azure Vault, etc)

[**Using Secret Managers with LiteLLM Proxy**](../secret)


### Set Supported Environments for a model - `production`, `staging`, `development`

Use this if you want to control which model is exposed on a specific litellm environment

Supported Environments:
- `production`
- `staging`
- `development`

1. Set `LITELLM_ENVIRONMENT="<environment>"` in your environment. Can be one of `production`, `staging` or `development`


2. For each model set the list of supported environments in `model_info.supported_environments`
```yaml
model_list:
 - model_name: gpt-3.5-turbo
   litellm_params:
     model: openai/gpt-3.5-turbo
     api_key: os.environ/OPENAI_API_KEY
   model_info:
     supported_environments: ["development", "production", "staging"]
 - model_name: gpt-4
   litellm_params:
     model: openai/gpt-4
     api_key: os.environ/OPENAI_API_KEY
   model_info:
     supported_environments: ["production", "staging"]
 - model_name: gpt-4o
   litellm_params:
     model: openai/gpt-4o
     api_key: os.environ/OPENAI_API_KEY
   model_info:
     supported_environments: ["production"]
```


### Set Custom Prompt Templates

LiteLLM by default checks if a model has a [prompt template and applies it](../completion/prompt_formatting.md) (e.g. if a huggingface model has a saved chat template in it's tokenizer_config.json). However, you can also set a custom prompt template on your proxy in the `config.yaml`: 

**Step 1**: Save your prompt template in a `config.yaml`
```yaml
# Model-specific parameters
model_list:
  - model_name: mistral-7b # model alias
    litellm_params: # actual params for litellm.completion()
      model: "huggingface/mistralai/Mistral-7B-Instruct-v0.1" 
      api_base: "<your-api-base>"
      api_key: "<your-api-key>" # [OPTIONAL] for hf inference endpoints
      initial_prompt_value: "\n"
      roles: {"system":{"pre_message":"<|im_start|>system\n", "post_message":"<|im_end|>"}, "assistant":{"pre_message":"<|im_start|>assistant\n","post_message":"<|im_end|>"}, "user":{"pre_message":"<|im_start|>user\n","post_message":"<|im_end|>"}}
      final_prompt_value: "\n"
      bos_token: " "
      eos_token: " "
      max_tokens: 4096
```

**Step 2**: Start server with config

```shell
$ litellm --config /path/to/config.yaml
``` 

## General Settings `general_settings` (DB Connection, etc)

### Configure DB Pool Limits + Connection Timeouts 

```yaml
general_settings: 
  database_connection_pool_limit: 100 # sets connection pool for prisma client to postgres db at 100
  database_connection_timeout: 60 # sets a 60s timeout for any connection call to the db 
```

## **All settings**


```yaml
environment_variables: {}

model_list:
  - model_name: string
    litellm_params: {}
    model_info:
      id: string
      mode: embedding
      input_cost_per_token: 0
      output_cost_per_token: 0
      max_tokens: 2048
      base_model: gpt-4-1106-preview
      additionalProp1: {}

litellm_settings:
  success_callback: ["langfuse"]  # list of success callbacks
  failure_callback: ["sentry"]  # list of failure callbacks
  callbacks: ["otel"]  # list of callbacks - runs on success and failure
  service_callbacks: ["datadog", "prometheus"]  # logs redis, postgres failures on datadog, prometheus
  turn_off_message_logging: boolean  # prevent the messages and responses from being logged to on your callbacks, but request metadata will still be logged.
  redact_user_api_key_info: boolean  # Redact information about the user api key (hashed token, user_id, team id, etc.), from logs. Currently supported for Langfuse, OpenTelemetry, Logfire, ArizeAI logging.

callback_settings:
  otel:
    message_logging: boolean  # OTEL logging callback specific settings

general_settings:
  completion_model: string
  disable_spend_logs: boolean  # turn off writing each transaction to the db
  disable_master_key_return: boolean  # turn off returning master key on UI (checked on '/user/info' endpoint)
  disable_retry_on_max_parallel_request_limit_error: boolean  # turn off retries when max parallel request limit is reached
  disable_reset_budget: boolean  # turn off reset budget scheduled task
  disable_adding_master_key_hash_to_db: boolean  # turn off storing master key hash in db, for spend tracking
  enable_jwt_auth: boolean  # allow proxy admin to auth in via jwt tokens with 'litellm_proxy_admin' in claims
  enforce_user_param: boolean  # requires all openai endpoint requests to have a 'user' param
  allowed_routes: ["route1", "route2"]  # list of allowed proxy API routes - a user can access. (currently JWT-Auth only)
  key_management_system: google_kms  # either google_kms or azure_kms
  master_key: string
  database_url: string
  database_connection_pool_limit: 0  # default 100
  database_connection_timeout: 0  # default 60s
  custom_auth: string
  max_parallel_requests: 0  # the max parallel requests allowed per deployment 
  global_max_parallel_requests: 0  # the max parallel requests allowed on the proxy all up 
  infer_model_from_keys: true
  background_health_checks: true
  health_check_interval: 300
  alerting: ["slack", "email"]
  alerting_threshold: 0
  use_client_credentials_pass_through_routes: boolean  # use client credentials for all pass through routes like "/vertex-ai", /bedrock/. When this is True Virtual Key auth will not be applied on these endpoints
```

### litellm_settings - Reference

| Name | Type | Description |
|------|------|-------------|
| success_callback | array of strings | List of success callbacks. [Doc Proxy logging callbacks](logging), [Doc Metrics](prometheus) |
| failure_callback | array of strings | List of failure callbacks [Doc Proxy logging callbacks](logging), [Doc Metrics](prometheus) |
| callbacks | array of strings | List of callbacks - runs on success and failure [Doc Proxy logging callbacks](logging), [Doc Metrics](prometheus) |
| service_callbacks | array of strings | System health monitoring - Logs redis, postgres failures on specified services (e.g. datadog, prometheus) [Doc Metrics](prometheus) |
| turn_off_message_logging | boolean | If true, prevents messages and responses from being logged to callbacks, but request metadata will still be logged [Proxy Logging](logging) |
| redact_user_api_key_info | boolean | If true, redacts information about the user api key from logs [Proxy Logging](logging#redacting-userapikeyinfo) |

### general_settings - Reference

| Name | Type | Description |
|------|------|-------------|
| completion_model | string | The default model to use for completions when `model` is not specified in the request |
| disable_spend_logs | boolean | If true, turns off writing each transaction to the database |
| disable_master_key_return | boolean | If true, turns off returning master key on UI. (checked on '/user/info' endpoint) |
| disable_retry_on_max_parallel_request_limit_error | boolean | If true, turns off retries when max parallel request limit is reached |
| disable_reset_budget | boolean | If true, turns off reset budget scheduled task |
| disable_adding_master_key_hash_to_db | boolean | If true, turns off storing master key hash in db |
| enable_jwt_auth | boolean | allow proxy admin to auth in via jwt tokens with 'litellm_proxy_admin' in claims. [Doc on JWT Tokens](token_auth) |
| enforce_user_param | boolean | If true, requires all OpenAI endpoint requests to have a 'user' param. [Doc on call hooks](call_hooks)|
| allowed_routes | array of strings | List of allowed proxy API routes a user can access [Doc on controlling allowed routes](enterprise#control-available-public-private-routes)|
| key_management_system | string | Specifies the key management system. [Doc Secret Managers](secret) |
| master_key | string | The master key for the proxy [Set up Virtual Keys](virtual_keys) |
| database_url | string | The URL for the database connection [Set up Virtual Keys](virtual_keys) |
| database_connection_pool_limit | integer | The limit for database connection pool [Setting DB Connection Pool limit](#configure-db-pool-limits--connection-timeouts) |
| database_connection_timeout | integer | The timeout for database connections in seconds [Setting DB Connection Pool limit, timeout](#configure-db-pool-limits--connection-timeouts) |
| custom_auth | string | Write your own custom authentication logic [Doc Custom Auth](virtual_keys#custom-auth) |
| max_parallel_requests | integer | The max parallel requests allowed per deployment |
| global_max_parallel_requests | integer | The max parallel requests allowed on the proxy overall |
| infer_model_from_keys | boolean | If true, infers the model from the provided keys |
| background_health_checks | boolean | If true, enables background health checks. [Doc on health checks](health) |
| health_check_interval | integer | The interval for health checks in seconds [Doc on health checks](health) |
| alerting | array of strings | List of alerting methods [Doc on Slack Alerting](alerting) |
| alerting_threshold | integer | The threshold for triggering alerts [Doc on Slack Alerting](alerting) |
| use_client_credentials_pass_through_routes | boolean | If true, uses client credentials for all pass-through routes. [Doc on pass through routes](pass_through) |

### router_settings - Reference

```yaml
router_settings:
  routing_strategy: usage-based-routing-v2 # Literal["simple-shuffle", "least-busy", "usage-based-routing","latency-based-routing"], default="simple-shuffle"
  redis_host: <your-redis-host>           # string
  redis_password: <your-redis-password>   # string
  redis_port: <your-redis-port>           # string
  enable_pre_call_check: true             # bool - Before call is made check if a call is within model context window 
  allowed_fails: 3 # cooldown model if it fails > 1 call in a minute. 
  cooldown_time: 30 # (in seconds) how long to cooldown model if fails/min > allowed_fails
  disable_cooldowns: True                  # bool - Disable cooldowns for all models 
  enable_tag_filtering: True                # bool - Use tag based routing for requests
  retry_policy: {                          # Dict[str, int]: retry policy for different types of exceptions
    "AuthenticationErrorRetries": 3,
    "TimeoutErrorRetries": 3,
    "RateLimitErrorRetries": 3,
    "ContentPolicyViolationErrorRetries": 4,
    "InternalServerErrorRetries": 4
  }
  allowed_fails_policy: {
    "BadRequestErrorAllowedFails": 1000, # Allow 1000 BadRequestErrors before cooling down a deployment
    "AuthenticationErrorAllowedFails": 10, # int 
    "TimeoutErrorAllowedFails": 12, # int 
    "RateLimitErrorAllowedFails": 10000, # int 
    "ContentPolicyViolationErrorAllowedFails": 15, # int 
    "InternalServerErrorAllowedFails": 20, # int 
  }
  content_policy_fallbacks=[{"claude-2": ["my-fallback-model"]}] # List[Dict[str, List[str]]]: Fallback model for content policy violations
  fallbacks=[{"claude-2": ["my-fallback-model"]}] # List[Dict[str, List[str]]]: Fallback model for all errors
```

| Name | Type | Description |
|------|------|-------------|
| routing_strategy | string | The strategy used for routing requests. Options: "simple-shuffle", "least-busy", "usage-based-routing", "latency-based-routing". Default is "simple-shuffle". [More information here](../routing) |
| redis_host | string | The host address for the Redis server. **Only set this if you have multiple instances of LiteLLM Proxy and want current tpm/rpm tracking to be shared across them** |
| redis_password | string | The password for the Redis server. **Only set this if you have multiple instances of LiteLLM Proxy and want current tpm/rpm tracking to be shared across them** |
| redis_port | string | The port number for the Redis server. **Only set this if you have multiple instances of LiteLLM Proxy and want current tpm/rpm tracking to be shared across them**|
| enable_pre_call_check | boolean | If true, checks if a call is within the model's context window before making the call. [More information here](reliability) |
| content_policy_fallbacks | array of objects | Specifies fallback models for content policy violations. [More information here](reliability) |
| fallbacks | array of objects | Specifies fallback models for all types of errors. [More information here](reliability) |
| enable_tag_filtering | boolean | If true, uses tag based routing for requests [Tag Based Routing](tag_routing) |
| cooldown_time | integer | The duration (in seconds) to cooldown a model if it exceeds the allowed failures. |
| disable_cooldowns | boolean | If true, disables cooldowns for all models. [More information here](reliability) |
| retry_policy | object | Specifies the number of retries for different types of exceptions. [More information here](reliability) |
| allowed_fails | integer | The number of failures allowed before cooling down a model. [More information here](reliability) |
| allowed_fails_policy | object | Specifies the number of allowed failures for different error types before cooling down a deployment. [More information here](reliability) |

## Extras


### Disable Swagger UI 

To disable the Swagger docs from the base url, set 

```env
NO_DOCS="True"
```

in your environment, and restart the proxy. 

### Use CONFIG_FILE_PATH for proxy (Easier Azure container deployment)

1. Setup config.yaml

```yaml
model_list:
  - model_name: gpt-3.5-turbo
    litellm_params:
      model: gpt-3.5-turbo
      api_key: os.environ/OPENAI_API_KEY
```

2. Store filepath as env var 

```bash
CONFIG_FILE_PATH="/path/to/config.yaml"
```

3. Start Proxy

```bash
$ litellm 

# RUNNING on http://0.0.0.0:4000
```


