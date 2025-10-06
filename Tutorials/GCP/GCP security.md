
### gcloud auth login command

The `gcloud auth login` command is a fundamental authentication tool in the Google Cloud SDK that authorizes the gcloud CLI to access Google Cloud Platform resources using your personal Google account credentials.[](https://cloud.google.com/sdk/gcloud/reference/auth/login)
#### Primary Function

The command obtains access credentials for your user account through a web-based OAuth 2.0 authorization flow. When executed successfully, it sets the active account in the current gcloud configuration to the specified account, and if no configuration exists, it creates a default configuration.[](https://prosperasoft.com/blog/cloud/gcp/gcloud-auth-login-vs-app-default/)
#### How It Works

Here’s a breakdown of what it does:

1. **Initiates Authentication:** When you run `gcloud auth login`, it opens a web browser and prompts you to log in to your Google Account (like a `@gmail.com` or Google Workspace account).
2. **Requests Permissions:** It will ask you to grant the "Google Cloud SDK" permission to access your account and manage your Google Cloud resources.
3. **Stores Credentials:** Once you approve, it receives an authorization token and stores it securely on your local machine.
4. **Activates Your Account:** It sets your user account as the "active" account for all subsequent `gcloud` commands you run in your terminal. For example, when you run `gcloud projects list`, it uses these credentials to fetch the list of projects your account has access to.

```bash
gcloud auth login [ACCOUNT] [FLAGS]
```
The `ACCOUNT` parameter is optional - if specified, it attempts to authorize that specific account.

#### Storage Locations by Operating System
The `gcloud auth login` command stores credentials in specific locations on your local machine that vary by operating system.

**Linux and macOS:**  
The credentials are stored in the `~/.config/gcloud` directory. Within this directory, the credentials are maintained in SQLite database files named `access_tokens.db` and `credentials.db`
##### Database Structure

The credentials are not stored as plain text files but rather in SQLite database format. The two main database files serve different purposes:[](https://stackoverflow.com/questions/40032678/where-are-google-application-default-credentials-stored)
- `credentials.db`: Contains the main credential information    
- `access_tokens.db`: Stores access tokens
##### **Distinguish `gcloud auth login` from `gcloud auth application-default login`**

##### 1. **gcloud CLI Authentication (`gcloud auth login`)**

- This is **only for the CLI tool itself**.    
- When you run:
    `gcloud auth login`
    you’re logging in with your Google account (or a service account).
- The CLI then stores your credentials locally (usually under `~/.config/gcloud/`).    
- From that point on, whenever you run `gcloud compute instances list` or `gcloud projects describe ...`, the CLI uses those credentials to talk to Google Cloud APIs.
##### 2. **Application Default Credentials (ADC) (`gcloud auth application-default login`)**

- ADC is **how your code (apps/scripts/libraries) authenticates to Google Cloud APIs**.    

- When you run:
    `gcloud auth application-default login`
    
    it asks you to log in, then generates a JSON file (typically stored at `~/.config/gcloud/application_default_credentials.json`).
        
- This file can then be picked up by Google SDKs (like Python’s `google-cloud-storage` library, or Terraform, etc.) when your code calls:
    
    `from google.cloud import storage client = storage.Client()`
    
    If the SDK sees that JSON file, it uses those credentials automatically.    

👉 Think of this as: _“I want my **applications/scripts** running on my machine to authenticate to GCP using my credentials.”_

##### 3. **Important Distinction**

- **CLI authentication config** = used only when _you_ run `gcloud ...` commands.    
- **ADC config** = used by _your code/tools/libraries_ when they need to authenticate to GCP.    
- They **don’t share the same state**, though you can log both in with the same account if you want.
    
For example:
- If you only run `gcloud auth login` and then try running a Python script that uses `google-cloud-storage`, the script may fail because it doesn’t have ADC credentials.    
- Conversely, if you only run `gcloud auth application-default login`, your `gcloud` CLI may still say _“you’re not logged in.”_

#### The `GOOGLE_APPLICATION_CREDENTIALS` Environment Variable

Application Default Credentials (ADC) is a strategy used by the authentication libraries to automatically find credentials based on the application environment.

When your Python code calls `google.auth.default()`, it follows a specific search order to find credentials. This is known as **Application Default Credentials (ADC)**. The order is:

1. **`GOOGLE_APPLICATION_CREDENTIALS` environment variable:** If this is set, its value (the path to the JSON key file) is used. **This takes top priority.**
2. **User credentials set by `gcloud auth application-default login`:** If the environment variable is _not_ set, ADC uses the credentials you created with this command.
3. **Attached service account:** When running on Google Cloud services like Compute Engine or Cloud Run, it uses the service account attached to that resource.

#### How GCP authentication works for RAG_BOT:

We've created a specific service account "rag_bot_local_dev" with limited permissions. A private key for this service account is created and stored locally at: 
"/home/bk_anupam/google-cloud-sdk/ardent-justice-466212-b8-3dde178b350e.json"

In the wsl /home/bk_anupam/.bashrc we set the environment variable using this line of code
export GOOGLE_APPLICATION_CREDENTIALS="/home/bk_anupam/google-cloud-sdk/ardent-justice-466212-b8-3dde178b350e.json" 

This ensures whenever code is run locally, if we need to authenticate to google cloud, the ADC uses the service account private key to authenticate and the code executes with the privileges granted to the service account.

