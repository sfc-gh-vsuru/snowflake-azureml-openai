# Snowflake Integration with Azure OpenAI using External Access

### Prerequisites

1. **This guide is a continuation from the below Quickstart Guide**

   **Note: Follow the QS guide from Step 1 to Step 3**
   Save the Azure Resource Name, API Key and model/deployment name

   [Quickstart Link](https://quickstarts.snowflake.com/guide/getting_started_with_azure_openai_streamlit_and_snowflake_for_image_use_cases/index.html?index=..%2F..index#0)

2. **Snowpark External Access to call Azure OpenAI - Step 4**

   **Note: Follow the below steps for Azure API using hosted OpenAI instead of Step 4 in the above Quickstart**

   Below script we would be creating External Access and UDF to make Azure API calls.
   Open a new SQL Worksheet in Snowsight, Paste the below code into the worksheet, update the values from your Azure account and click run

```sql
use role ACCOUNTADMIN;
use database RETAIL_HOL;
use warehouse HOL_WH;

CREATE OR REPLACE NETWORK RULE CHATGPT_NETWORK_RULE
    MODE = EGRESS
    TYPE = HOST_PORT
    VALUE_LIST = ('{your-resource-name}.openai.azure.com'); -- Update your Azure resource name from Step 2

CREATE OR REPLACE SECRET CHATGPT_API_KEY
    TYPE = GENERIC_STRING
    SECRET_STRING='YOUR_API_KEY'; -- Update your Azure API Key from Step 2

CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION OPENAI_INTEGRATION
    ALLOWED_NETWORK_RULES = (CHATGPT_NETWORK_RULE)
    ALLOWED_AUTHENTICATION_SECRETS = (CHATGPT_API_KEY)
    ENABLED=TRUE;

CREATE OR REPLACE FUNCTION CHATGPT_IMAGE(instructions STRING, list STRING, user_context STRING)
returns string
language python
runtime_version=3.8
handler = 'ask_chatGPT'
external_access_integrations=(OPENAI_INTEGRATION)
packages = ('openai')
SECRETS = ('cred' = chatgpt_api_key )
as
$$
import _snowflake
import json
from openai import AzureOpenAI
client = AzureOpenAI(
    api_key=_snowflake.get_generic_secret_string("cred"),
    api_version='2023-12-01-preview',
    # Update Resource and Model to the base_url below
    base_url="https://{your-resource-name}.openai.azure.com/openai/deployments/{model}/extensions"
    )
def ask_chatGPT(instructions, list_, user_context):
    response = client.chat.completions.create(
    model='{model}', # Update your model/deployment name from Step 2
    messages = [
        {
            "role": "system",
            "content": json.dumps({
                "SYSTEM": f"Follow these: {instructions}",
                "CONTEXT_LIST": f"Use this list to select from {list_}",
                "USER_CONTEXT": f"Use this image for your response: {user_context}"
            })
        }
    ],
    max_tokens=2000 )
    return response.choices[0].message.content
$$;

```

3. **Follow the Quickstart - Step 5 Build Streamlit App and the remaining steps in the QS**

   [Quickstart Link](https://quickstarts.snowflake.com/guide/getting_started_with_azure_openai_streamlit_and_snowflake_for_image_use_cases/index.html?index=..%2F..index#4)

### This Concludes setup and working with Snowflake External Access with Azure OpenAI
