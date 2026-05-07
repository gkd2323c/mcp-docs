# MCP Documentation -- 09 Registry

- The MCP Registry
- How to Authenticate When Publishing to the Official MCP Registry
- Frequently Asked Questions
- How to Automate Publishing with GitHub Actions
- The MCP Registry Moderation Policy
- MCP Registry Supported Package Types
- Package Types
- Database Query MCP Server
- Azure DevOps MCP Server
- Quickstart: Publish an MCP Server to the MCP Registry
- Navigate to project directory
- Install dependencies
- Build the distribution files
- If necessary, authenticate to npm
- Publish the package
- MCP Registry Aggregators
- Publishing Remote Servers
- Official MCP Registry Terms of Service
- Versioning Published MCP Servers

---

# The MCP Registry
Source: https://modelcontextprotocol.io/registry/about

  The MCP Registry is currently in preview. Breaking changes or data resets may occur before general availability. If you encounter any issues, please report them on [GitHub](https://github.com/modelcontextprotocol/registry/issues).

The MCP Registry is the official centralized metadata repository for publicly accessible MCP servers, backed by major trusted contributors to the MCP ecosystem such as Anthropic, GitHub, PulseMCP, and Microsoft.

The MCP Registry provides:

* A single place for server creators to publish metadata about their servers
* Namespace management through DNS verification
* A REST API for MCP clients and aggregators to discover available servers
* Standardized installation and configuration information

Server metadata is stored in a standardized [`server.json` format](https://github.com/modelcontextprotocol/registry/blob/main/docs/reference/server-json/draft/server.schema.json), which contains:

* The server's unique name (e.g., `io.github.user/server-name`)
* Where to locate the server (e.g., npm package name, remote server URL)
* Execution instructions (e.g., command-line args, env vars)
* Other discovery data (e.g., description, server capabilities)

## The MCP Registry Ecosystem

The MCP Registry is part of an ecosystem that looks something like:

[Image: The MCP Registry ecosystem]

### Relationship with Package Registries

Package registries — such as npm, PyPI, and Docker Hub — host packages with code and binaries.

The MCP Registry hosts metadata that points to those packages.

For example, a `weather-mcp` package could be hosted on npm, and metadata in the MCP Registry could map the "weather v1.2.0" server to `npm:weather-mcp`.

The [Package Types guide](./package-types) lists the supported package types and registries. More package registries may be supported in the future based on community demand. If you are interested in building support for a package registry, please [open an issue](https://github.com/modelcontextprotocol/registry).

### Relationship with Server Developers

The MCP Registry supports both open-source and closed-source servers. Server developers can publish their server's metadata to the registry as long as the server's installation method is publicly available (e.g., an npm package or a Docker image on a public registry) *or* the server itself is publicly accessible (e.g., a remote server that is not restricted to private networks).

The MCP Registry **does not** support private servers. Private servers are those that are only accessible to a narrow set of users. For example, servers published on a private network (like `mcp.acme-corp.internal`) or on private package registries (e.g. `npx -y @acme/mcp --registry https://artifactory.acme-corp.internal/npm`). If you want to publish private servers, we recommend that you host your own private MCP registry and add them there.

### Relationship with Downstream Aggregators

The MCP Registry is intended to be consumed primarily by downstream aggregators, such as MCP server marketplaces.

The metadata hosted by the MCP Registry is deliberately unopinionated. Downstream aggregators can provide curation or additional metadata such as community ratings.

We expect that downstream aggregators will use the MCP Registry API to pull new metadata on a regular but infrequent basis (for example, once per hour). See the [MCP Registry Aggregators guide](./registry-aggregators) for more information.

### Relationship with Other MCP Registries

In addition to a public REST API, the MCP Registry defines an [OpenAPI spec](https://github.com/modelcontextprotocol/registry/blob/main/docs/reference/api/openapi.yaml) that other MCP registries can implement in order to provide a standardized interface for MCP host applications.

We expect that many downstream aggregators will implement this interface. Private MCP registries can implement it as well to benefit from existing host application support.

Note that the official MCP Registry codebase is **not** designed for self-hosting, and the registry maintainers cannot provide support for this use case. If you choose to fork it, you would need to maintain and operate it independently.

### Relationship with MCP Host Applications

The MCP Registry is not intended to be directly consumed by host applications. Instead, host applications should consume other MCP registries, such as downstream marketplaces, via a REST API conforming to the official MCP Registry's OpenAPI spec.

## Trust and Security

### Verifying Server Authenticity

The MCP Registry uses namespace authentication to ensure that servers come from their claimed sources. Server names follow a reverse DNS format (like `io.github.username/server` or `com.example/server`) that ties them to verified GitHub accounts or domains.

This namespace system ensures that only the legitimate owner of a GitHub account or domain can publish servers under that namespace, providing trust and accountability in the ecosystem. For details on authentication methods, see the [Authentication guide](./authentication).

### Security Scanning

The MCP Registry delegates security scanning to:

* **Underlying package registries** — npm, PyPI, Docker Hub, and other package registries perform their own security scanning and vulnerability detection.
* **Downstream aggregators** — MCP Registry aggregators and marketplaces can implement additional security checks, ratings, or curation.

The MCP Registry focuses on namespace authentication and metadata hosting, while relying on the broader ecosystem for security scanning of actual server code.

### Spam Prevention

The MCP Registry uses multiple mechanisms to prevent spam:

* **Namespace authentication requirements** — Publishers must verify ownership of their namespace through GitHub, DNS, or HTTP challenges, preventing arbitrary spam submissions.
* **Character limits and validation** — Free-form fields have strict character limits and regex validation to prevent abuse.
* **Manual takedown** — The registry maintainers can manually remove spam or malicious servers. See the [Moderation Policy](./moderation-policy) for details on what content is removed.

Future spam prevention measures under consideration include stricter rate limiting, AI-based spam detection, and community reporting capabilities.

# How to Authenticate When Publishing to the Official MCP Registry
Source: https://modelcontextprotocol.io/registry/authentication

  The MCP Registry is currently in preview. Breaking changes or data resets may occur before general availability. If you encounter any issues, please report them on [GitHub](https://github.com/modelcontextprotocol/registry/issues).

You must authenticate before publishing to the official MCP Registry. The MCP Registry supports different authentication methods. Which authentication method you choose determines the namespace of your server's name.

If you choose GitHub-based authentication, your server's name in `server.json` **MUST** be of the form `io.github.username/*` (or `io.github.orgname/*`). For example, `io.github.alice/weather-server`.

If you choose domain-based authentication, your server's name in `server.json` **MUST** be of the form `com.example.*/*`, where `com.example` is the reverse-DNS form of your domain name. For example, `io.modelcontextprotocol/everything`.

| Authentication | Name Format                                     | Example Name                         |
| -------------- | ----------------------------------------------- | ------------------------------------ |
| GitHub-based   | `io.github.username/*` or `io.github.orgname/*` | `io.github.alice/weather-server`     |
| domain-based   | `com.example.*/*`                               | `io.modelcontextprotocol/everything` |

## GitHub Authentication

GitHub authentication uses an OAuth flow initiated by the `mcp-publisher` CLI tool.

To perform GitHub authentication, navigate to your server project directory and run:

```bash 
mcp-publisher login github
```

You should see output like:

```text Output 
Logging in with github...

To authenticate, please:
1. Go to: https://github.com/login/device
2. Enter code: ABCD-1234
3. Authorize this application
Waiting for authorization...
```

Visit the link, follow the prompts, and enter the authorization code that was printed in the terminal (e.g., `ABCD-1234` in the above output). Once complete, go back to the terminal, and you should see output like:

```text Output 
Successfully authenticated!
✓ Successfully logged in
```

## DNS Authentication

DNS authentication is a domain-based authentication method that relies on a DNS TXT record.

To perform DNS authentication using the `mcp-publisher` CLI tool, run the following commands in your server project directory to generate a TXT record based on a public/private key pair:

  ```bash Ed25519 
  MY_DOMAIN="example.com"

  # Generate public/private key pair using Ed25519
  openssl genpkey -algorithm Ed25519 -out key.pem

  # Generate TXT record
  PUBLIC_KEY="$(openssl pkey -in key.pem -pubout -outform DER | tail -c 32 | base64)"
  echo "${MY_DOMAIN}. IN TXT \"v=MCPv1; k=ed25519; p=${PUBLIC_KEY}\""
  ```

  ```bash ECDSA P-384 
  MY_DOMAIN="example.com"

  # Generate public/private key pair using ECDSA P-384
  openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:secp384r1 -out key.pem

  # Generate TXT record
  PUBLIC_KEY="$(openssl ec -in key.pem -text -noout -conv_form compressed | grep -A4 "pub:" | tail -n +2 | tr -d ' :\n' | xxd -r -p | base64)"
  echo "${MY_DOMAIN}. IN TXT \"v=MCPv1; k=ecdsap384; p=${PUBLIC_KEY}\""
  ```

  ```bash Google KMS 
  MY_DOMAIN="example.com"
  MY_PROJECT="myproject"
  MY_KEYRING="mykeyring"
  MY_KEY_NAME="mykey"

  # Log in using gcloud CLI (https://cloud.google.com/sdk/docs/install)
  gcloud auth login

  # Set default project
  gcloud config set project "${MY_PROJECT}"

  # Create a keyring in your project
  gcloud kms keyrings create "${MY_KEYRING}" --location global

  # Create an Ed25519 signing key
  gcloud kms keys create "${MY_KEY_NAME}" --default-algorithm=ec-sign-ed25519 --purpose=asymmetric-signing --keyring="${MY_KEYRING}" --location=global

  # Enable Application Default Credentials (ADC) so the publisher tool can sign
  gcloud auth application-default login

  # Attempt login to show the public key
  mcp-publisher login dns google-kms --domain="${MY_DOMAIN}" --resource="projects/${MY_PROJECT}/locations/global/keyRings/${MY_KEYRING}/cryptoKeys/${MY_KEY_NAME}/cryptoKeyVersions/1"

  # Copy the "Expected proof record":
  # ${MY_DOMAIN}. IN TXT "v=MCPv1; k=ed25519; p=${PUBLIC_KEY}"
  ```

  ```bash Azure Key Vault 
  MY_DOMAIN="example.com"
  MY_SUBSCRIPTION="subscription name or ID"
  MY_RESOURCE_GROUP="MyResourceGroup"
  MY_KEY_VAULT="MyKeyVault"
  MY_KEY_NAME="MyKey"

  # Log in using Azure CLI (https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
  az login

  # Set default subscription
  az account set --subscription "${MY_SUBSCRIPTION}"

  # Create a resource group
  az group create --location westus --resource-group "${MY_RESOURCE_GROUP}"

  # Create a key vault
  az keyvault create --name "${MY_KEY_VAULT}" --location westus --resource-group "${MY_RESOURCE_GROUP}"

  # Create an ECDSA P-384 signing key
  az keyvault key create --name "${MY_KEY_NAME}" --vault-name "${MY_KEY_VAULT}" --curve P-384

  # Attempt login to show the public key
  mcp-publisher login dns azure-key-vault --domain="${MY_DOMAIN}" --vault "${MY_KEY_VAULT}" --key "${MY_KEY_NAME}"

  # Copy the "Expected proof record":
  # ${MY_DOMAIN}. IN TXT "v=MCPv1; k=ecdsap384; p=${PUBLIC_KEY}"
  ```

Then add the TXT record using your DNS provider's control panel. It may take several minutes for the TXT record to propagate. After the TXT record has propagated, log in using the `mcp-publisher login` command:

  ```bash Ed25519 
  MY_DOMAIN="example.com"

  PRIVATE_KEY="$(openssl pkey -in key.pem -noout -text | grep -A3 "priv:" | tail -n +2 | tr -d ' :\n')"
  mcp-publisher login dns --domain "${MY_DOMAIN}" --private-key "${PRIVATE_KEY}"
  ```

  ```bash ECDSA P-384 
  MY_DOMAIN="example.com"

  PRIVATE_KEY="$(openssl ec -in key.pem -noout -text | grep -A4 "priv:" | tail -n +2 | tr -d ' :\n')"
  mcp-publisher login dns --domain "${MY_DOMAIN}" --private-key "${PRIVATE_KEY}"
  ```

  ```bash Google KMS 
  MY_DOMAIN="example.com"
  MY_PROJECT="myproject"
  MY_KEYRING="mykeyring"
  MY_KEY_NAME="mykey"

  mcp-publisher login dns google-kms --domain="${MY_DOMAIN}" --resource="projects/${MY_PROJECT}/locations/global/keyRings/${MY_KEYRING}/cryptoKeys/${MY_KEY_NAME}/cryptoKeyVersions/1"
  ```

  ```bash Azure Key Vault 
  MY_DOMAIN="example.com"
  MY_KEY_VAULT="MyKeyVault"
  MY_KEY_NAME="MyKey"

  mcp-publisher login dns azure-key-vault --domain="${MY_DOMAIN}" --vault "${MY_KEY_VAULT}" --key "${MY_KEY_NAME}"
  ```

## HTTP Authentication

HTTP authentication is a domain-based authentication method that relies on a `/.well-known/mcp-registry-auth` file hosted on your domain. For example, `https://example.com/.well-known/mcp-registry-auth`.

To perform HTTP authentication using the `mcp-publisher` CLI tool, run the following commands in your server project directory to generate an `mcp-registry-auth` file based on a public/private key pair:

  ```bash Ed25519 
  # Generate public/private key pair using Ed25519
  openssl genpkey -algorithm Ed25519 -out key.pem

  # Generate mcp-registry-auth file
  PUBLIC_KEY="$(openssl pkey -in key.pem -pubout -outform DER | tail -c 32 | base64)"
  echo "v=MCPv1; k=ed25519; p=${PUBLIC_KEY}" > mcp-registry-auth
  ```

  ```bash ECDSA P-384 
  # Generate public/private key pair using ECDSA P-384
  openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:secp384r1 -out key.pem

  # Generate mcp-registry-auth file
  PUBLIC_KEY="$(openssl ec -in key.pem -text -noout -conv_form compressed | grep -A4 "pub:" | tail -n +2 | tr -d ' :\n' | xxd -r -p | base64)"
  echo "v=MCPv1; k=ecdsap384; p=${PUBLIC_KEY}" > mcp-registry-auth
  ```

  ```bash Google KMS 
  MY_DOMAIN="example.com"
  MY_PROJECT="myproject"
  MY_KEYRING="mykeyring"
  MY_KEY_NAME="mykey"

  # Log in using gcloud CLI (https://cloud.google.com/sdk/docs/install)
  gcloud auth login

  # Set default project
  gcloud config set project "${MY_PROJECT}"

  # Create a keyring in your project
  gcloud kms keyrings create "${MY_KEYRING}" --location global

  # Create an Ed25519 signing key
  gcloud kms keys create "${MY_KEY_NAME}" --default-algorithm=ec-sign-ed25519 --purpose=asymmetric-signing --keyring="${MY_KEYRING}" --location=global

  # Enable Application Default Credentials (ADC) so the publisher tool can sign
  gcloud auth application-default login

  # Attempt login to show the public key
  mcp-publisher login http google-kms --domain="${MY_DOMAIN}" --resource="projects/${MY_PROJECT}/locations/global/keyRings/${MY_KEYRING}/cryptoKeys/${MY_KEY_NAME}/cryptoKeyVersions/1"

  # Copy the "Expected proof record" to `./mcp-registry-auth`:
  # v=MCPv1; k=ed25519; p=${PUBLIC_KEY}
  ```

  ```bash Azure Key Vault 
  MY_DOMAIN="example.com"
  MY_SUBSCRIPTION="subscription name or ID"
  MY_RESOURCE_GROUP="MyResourceGroup"
  MY_KEY_VAULT="MyKeyVault"
  MY_KEY_NAME="MyKey"

  # Log in using Azure CLI (https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
  az login

  # Set default subscription
  az account set --subscription "${MY_SUBSCRIPTION}"

  # Create a resource group
  az group create --location westus --resource-group "${MY_RESOURCE_GROUP}"

  # Create a key vault
  az keyvault create --name "${MY_KEY_VAULT}" --location westus --resource-group "${MY_RESOURCE_GROUP}"

  # Create an ECDSA P-384 signing key
  az keyvault key create --name "${MY_KEY_NAME}" --vault-name "${MY_KEY_VAULT}" --curve P-384

  # Attempt login to show the public key
  mcp-publisher login http azure-key-vault --domain="${MY_DOMAIN}" --vault "${MY_KEY_VAULT}" --key "${MY_KEY_NAME}"

  # Copy the "Expected proof record" to `./mcp-registry-auth`:
  # v=MCPv1; k=ecdsap384; p=${PUBLIC_KEY}
  ```

Then host the `mcp-registry-auth` file at `/.well-known/mcp-registry-auth` on your domain. After the file is hosted, log in using the `mcp-publisher login` command:

  ```bash Ed25519 
  MY_DOMAIN="example.com"
  PRIVATE_KEY="$(openssl pkey -in key.pem -noout -text | grep -A3 "priv:" | tail -n +2 | tr -d ' :\n')"
  mcp-publisher login http --domain "${MY_DOMAIN}" --private-key "${PRIVATE_KEY}"
  ```

  ```bash ECDSA P-384 
  MY_DOMAIN="example.com"
  PRIVATE_KEY="$(openssl ec -in key.pem -noout -text | grep -A4 "priv:" | tail -n +2 | tr -d ' :\n')"
  mcp-publisher login http --domain "${MY_DOMAIN}" --private-key "${PRIVATE_KEY}"
  ```

  ```bash Google KMS 
  MY_DOMAIN="example.com"
  MY_PROJECT="myproject"
  MY_KEYRING="mykeyring"
  MY_KEY_NAME="mykey"

  mcp-publisher login http google-kms --domain="${MY_DOMAIN}" --resource="projects/${MY_PROJECT}/locations/global/keyRings/${MY_KEYRING}/cryptoKeys/${MY_KEY_NAME}/cryptoKeyVersions/1"
  ```

  ```bash Azure Key Vault 
  MY_DOMAIN="example.com"
  MY_KEY_VAULT="MyKeyVault"
  MY_KEY_NAME="MyKey"

  mcp-publisher login http azure-key-vault --domain="${MY_DOMAIN}" --vault "${MY_KEY_VAULT}" --key "${MY_KEY_NAME}"
  ```

# Frequently Asked Questions
Source: https://modelcontextprotocol.io/registry/faq

  The MCP Registry is currently in preview. Breaking changes or data resets may occur before general availability. If you encounter any issues, please report them on [GitHub](https://github.com/modelcontextprotocol/registry/issues).

## General

### What is the difference between "Official MCP Registry", "MCP Registry", "MCP registry", "MCP Registry API", etc?

* "MCP Registry API" — An API that implements the [OpenAPI spec](https://github.com/modelcontextprotocol/registry/blob/main/docs/reference/api/openapi.yaml) defined by the MCP Registry.
* "Official MCP Registry API" — The REST API served at `https://registry.modelcontextprotocol.io`, which is a superset of the MCP Registry API. Its OpenAPI spec can be downloaded from [https://registry.modelcontextprotocol.io/openapi.yaml](https://registry.modelcontextprotocol.io/openapi.yaml).
* "MCP registry" — A third-party service that provides an MCP Registry API.
* "Official MCP Registry" (or "The MCP Registry") — The service that lives at `https://registry.modelcontextprotocol.io`.

### Can I delete/unpublish my server?

Currently, no. At the time of writing, there is [open discussion](https://github.com/modelcontextprotocol/registry/issues/104).

### How do I update my server metadata?

Submit a new `server.json` with a unique version string. Once published, version metadata is immutable (similar to npm).

### Can I add custom metadata when publishing?

Yes, custom metadata under `_meta.io.modelcontextprotocol.registry/publisher-provided` is preserved when publishing to the registry. This allows you to include custom metadata specific to your publishing process.

  There is a 4KB size limit (4096 bytes of JSON). Publishing will fail if this limit is exceeded.

## Reporting Issues

### What if I need to report a spam or malicious server?

1. Report it as abuse to the underlying package registry (e.g. NPM, PyPi, DockerHub, etc.); and
2. Raise a GitHub issue on the registry repo with a title beginning `Abuse report: `

### What if I need to report a security vulnerability in the registry itself?

Follow [the MCP community SECURITY.md](https://github.com/modelcontextprotocol/.github/blob/main/SECURITY.md).

# How to Automate Publishing with GitHub Actions
Source: https://modelcontextprotocol.io/registry/github-actions

  The MCP Registry is currently in preview. Breaking changes or data resets may occur before general availability. If you encounter any issues, please report them on [GitHub](https://github.com/modelcontextprotocol/registry/issues).

## Step 1: Create a Workflow File

In your server project directory, create a `.github/workflows/publish-mcp.yml` file. Here is an example for npm-based local server, but the MCP Registry publishing steps are the same for all package types:

  ```yaml OIDC authentication (recommended) 
  name: Publish to MCP Registry

  on:
    push:
      tags: ["v*"] # Triggers on version tags like v1.0.0

  jobs:
    publish:
      runs-on: ubuntu-latest
      permissions:
        id-token: write # Required for OIDC authentication
        contents: read

      steps:
        - name: Checkout code
          uses: actions/checkout@v5

        ### Publish underlying npm package:

        - name: Set up Node.js
          uses: actions/setup-node@v5
          with:
            node-version: "lts/*"

        - name: Install dependencies
          run: npm ci

        - name: Run tests
          run: npm run test --if-present

        - name: Build package
          run: npm run build --if-present

        - name: Publish package to npm
          run: npm publish
          env:
            NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}

        ### Publish MCP server:

        - name: Install mcp-publisher
          run: |
            curl -L "https://github.com/modelcontextprotocol/registry/releases/latest/download/mcp-publisher_$(uname -s | tr '[:upper:]' '[:lower:]')_$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/').tar.gz" | tar xz mcp-publisher

        - name: Authenticate to MCP Registry
          run: ./mcp-publisher login github-oidc

        # Optional:
        # - name: Set version in server.json
        #   run: |
        #     VERSION=${GITHUB_REF#refs/tags/v}
        #     jq --arg v "$VERSION" '.version = $v' server.json > server.tmp && mv server.tmp server.json

        - name: Publish server to MCP Registry
          run: ./mcp-publisher publish
  ```

  ```yaml PAT authentication 
  name: Publish to MCP Registry

  on:
    push:
      tags: ["v*"] # Triggers on version tags like v1.0.0

  jobs:
    publish:
      runs-on: ubuntu-latest
      permissions:
        contents: read

      steps:
        - name: Checkout code
          uses: actions/checkout@v5

        ### Publish underlying npm package:

        - name: Set up Node.js
          uses: actions/setup-node@v5
          with:
            node-version: "lts/*"

        - name: Install dependencies
          run: npm ci

        - name: Run tests
          run: npm run test --if-present

        - name: Build package
          run: npm run build --if-present

        - name: Publish package to npm
          run: npm publish
          env:
            NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}

        ### Publish MCP server:

        - name: Install mcp-publisher
          run: |
            curl -L "https://github.com/modelcontextprotocol/registry/releases/latest/download/mcp-publisher_$(uname -s | tr '[:upper:]' '[:lower:]')_$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/').tar.gz" | tar xz mcp-publisher

        - name: Authenticate to MCP Registry
          run: ./mcp-publisher login github --token ${{ secrets.MCP_GITHUB_TOKEN }}

        # Optional:
        # - name: Set version in server.json
        #   run: |
        #     VERSION=${GITHUB_REF#refs/tags/v}
        #     jq --arg v "$VERSION" '.version = $v' server.json > server.tmp && mv server.tmp server.json

        - name: Publish server to MCP Registry
          run: ./mcp-publisher publish
  ```

  ```yaml DNS authentication 
  name: Publish to MCP Registry

  on:
    push:
      tags: ["v*"] # Triggers on version tags like v1.0.0

  jobs:
    publish:
      runs-on: ubuntu-latest
      permissions:
        contents: read

      steps:
        - name: Checkout code
          uses: actions/checkout@v5

        ### Publish underlying npm package:

        - name: Set up Node.js
          uses: actions/setup-node@v5
          with:
            node-version: "lts/*"

        - name: Install dependencies
          run: npm ci

        - name: Run tests
          run: npm run test --if-present

        - name: Build package
          run: npm run build --if-present

        - name: Publish package to npm
          run: npm publish
          env:
            NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}

        ### Publish MCP server:

        - name: Install mcp-publisher
          run: |
            curl -L "https://github.com/modelcontextprotocol/registry/releases/latest/download/mcp-publisher_$(uname -s | tr '[:upper:]' '[:lower:]')_$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/').tar.gz" | tar xz mcp-publisher

        # !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
        # TODO: Replace `example.com` with your domain name
        # !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
        - name: Authenticate to MCP Registry
          run: ./mcp-publisher login dns --domain example.com --private-key ${{ secrets.MCP_PRIVATE_KEY }}

        # Optional:
        # - name: Set version in server.json
        #   run: |
        #     VERSION=${GITHUB_REF#refs/tags/v}
        #     jq --arg v "$VERSION" '.version = $v' server.json > server.tmp && mv server.tmp server.json

        - name: Publish server to MCP Registry
          run: ./mcp-publisher publish
  ```

## Step 2: Add Secrets

You may need to add a secret to the repository depending on which authentication method you choose:

* **GitHub OIDC Authentication**: No dedicated secret necessary.
* **GitHub PAT Authentication**: Add a `MCP_GITHUB_TOKEN` secret with a GitHub Personal Access Token (PAT) that has `read:org` and `read:user` scopes.
* **DNS Authentication**: Add a `MCP_PRIVATE_KEY` secret with your Ed25519 private key.

You may also need to add secrets for your package registry. For example, the workflow above needs an `NPM_TOKEN` secret with your npm token.

For information about how to add secrets to a repository, see [Using secrets in GitHub Actions](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets).

## Step 3: Tag and Release

Create and push a version tag to trigger the workflow:

```bash 
git tag v1.0.0
git push origin v1.0.0
```

The workflow will run tests, build the package, publish the package to npm, and publish the server to the MCP Registry.

## Troubleshooting

| Error Message               | Action                                                                                                                                                                     |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| "Authentication failed"     | Ensure `id-token: write` permission is set for OIDC, or check secrets.                                                                                                     |
| "Package validation failed" | Verify your package successfully published to the package registry (e.g., npm, PyPI), and that your package has the [necessary verification information](./package-types). |

# The MCP Registry Moderation Policy
Source: https://modelcontextprotocol.io/registry/moderation-policy

  The MCP Registry is currently in preview. Breaking changes or data resets may occur before general availability. If you encounter any issues, please report them on [GitHub](https://github.com/modelcontextprotocol/registry/issues).

**TL;DR**: The MCP Registry is quite permissive! We only remove illegal content, malware, spam, and completely broken servers.

## Scope

This policy applies to the official MCP Registry at `registry.modelcontextprotocol.io`.

Subregistries may have their own moderation policies. If you have questions about content on a specific subregistry, please contact them directly.

## Disclaimer

The MCP Registry **does not** make guarantees about moderation, and consumers should assume minimal-to-no moderation.

The MCP Registry is a community supported project, and we have limited active moderation capabilities. We largely rely on upstream package registries (like NPM, PyPI, and Docker) or downstream subregistries (like the GitHub MCP Registry) to do more in-depth moderation.

This means there may be content in the MCP Registry that should be removed under this policy, but which we haven't yet removed. Consumers should treat scraped data accordingly.

## What We Remove

We will remove servers that contain:

* Illegal content, which includes obscene content, copyright violations, and hacking tools
* Malware, regardless of intentions
* Spam, especially mass-created servers that disrupt the registry. Examples:
  * The same server being submitted multiple times under different names
  * A server that doesn't do anything but provide a fixed response with some marketing copy
  * A server with a description stuffed with marketing copy and an unrelated implementation
* Non-functioning servers

## What We Don't Remove

Generally, we believe in keeping the registry open and pushing moderation to subregistries. We therefore **won't** remove:

* Low-quality or buggy servers
* Servers with security vulnerabilities
* Servers that do the same thing as other servers
* Servers that provide or contain adult content

## How Removal Works

When we remove a server, we set the server's `status` to `"deleted"`, but the server's metadata remains accessible via the MCP Registry API. Aggregators may then remove the server from their indexes.

In extreme cases, we may overwrite or erase the server's metadata. For example, if the metadata itself is unlawful.

## Appeals

Think we made a mistake? Open an issue on our [GitHub repository](https://github.com/modelcontextprotocol/registry) with:

* The name of the server
* Why you believe the server doesn't meet the above criteria for removal

## Changes to This Policy

We're still learning how best to run the MCP Registry! As such, we might end up changing this policy in the future.

# MCP Registry Supported Package Types
Source: https://modelcontextprotocol.io/registry/package-types

  The MCP Registry is currently in preview. Breaking changes or data resets may occur before general availability. If you encounter any issues, please report them on [GitHub](https://github.com/modelcontextprotocol/registry/issues).

# Package Types

The MCP Registry supports several different package types, and each package type has its own verification method.

## npm Packages

For npm packages, the MCP Registry currently supports the npm public registry (`https://registry.npmjs.org`) only.

npm packages use `"registryType": "npm"` in `server.json`. For example:

```json server.json highlight={9} 
{
  "$schema": "https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json",
  "name": "io.github.username/email-integration-mcp",
  "title": "Email Integration",
  "description": "Send emails and manage email accounts",
  "version": "1.0.0",
  "packages": [
    {
      "registryType": "npm",
      "identifier": "@username/email-integration-mcp",
      "version": "1.0.0",
      "transport": {
        "type": "stdio"
      }
    }
  ]
}
```

### Ownership Verification

The MCP Registry verifies ownership of npm packages by checking `mcpName` in `package.json`. The `mcpName` property **MUST** match the server name from `server.json`. For example:

```json package.json 
{
  "name": "@username/email-integration-mcp",
  "version": "1.0.0",
  "mcpName": "io.github.username/email-integration-mcp"
}
```

## PyPI Packages

For PyPI packages, the MCP Registry currently supports the official PyPI registry (`https://pypi.org`) only.

PyPI packages use `"registryType": "pypi"` in `server.json`. For example:

```json server.json highlight={9} 
{
  "$schema": "https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json",
  "name": "io.github.username/database-query-mcp",
  "title": "Database Query",
  "description": "Execute SQL queries and manage database connections",
  "version": "1.0.0",
  "packages": [
    {
      "registryType": "pypi",
      "identifier": "database-query-mcp",
      "version": "1.0.0",
      "transport": {
        "type": "stdio"
      }
    }
  ]
}
```

### Ownership Verification

The MCP Registry verifies ownership of PyPI packages by checking for the existence of an `mcp-name: $SERVER_NAME` string in the package README (which becomes the package description on PyPI). The string may be hidden in a comment, but the `$SERVER_NAME` portion **MUST** match the server name from `server.json`. For example:

```markdown README.md highlight={5} 
# Database Query MCP Server

This MCP server executes SQL queries and manages database connections.

<!-- mcp-name: io.github.username/database-query-mcp -->
```

## NuGet Packages

For NuGet packages, the MCP Registry currently supports the official NuGet registry (`https://api.nuget.org/v3/index.json`) only.

NuGet packages use `"registryType": "nuget"` in `server.json`. For example:

```json server.json highlight={9} 
{
  "$schema": "https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json",
  "name": "io.github.username/azure-devops-mcp",
  "title": "Azure DevOps",
  "description": "Manage Azure DevOps work items and pipelines",
  "version": "1.0.0",
  "packages": [
    {
      "registryType": "nuget",
      "identifier": "Username.AzureDevOpsMcp",
      "version": "1.0.0",
      "transport": {
        "type": "stdio"
      }
    }
  ]
}
```

### Ownership Verification

The MCP Registry verifies ownership of NuGet packages by checking for the existence of an `mcp-name: $SERVER_NAME` string in the package README. The string may be hidden in a comment, but the `$SERVER_NAME` portion **MUST** match the server name from `server.json`. For example:

```markdown README.md highlight={5} 
# Azure DevOps MCP Server

This MCP server manages Azure DevOps work items and pipelines.

<!-- mcp-name: io.github.username/azure-devops-mcp -->
```

## Docker/OCI Images

For Docker/OCI images, the MCP Registry currently supports:

* Docker Hub (`docker.io`)
* GitHub Container Registry (`ghcr.io`)
* Google Artifact Registry (any `*.pkg.dev` domain)
* Azure Container Registry (`*.azurecr.io`)
* Microsoft Container Registry (`mcr.microsoft.com`)

Docker/OCI images use `"registryType": "oci"` in `server.json`. For example:

```json server.json highlight={9} 
{
  "$schema": "https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json",
  "name": "io.github.username/kubernetes-manager-mcp",
  "title": "Kubernetes Manager",
  "description": "Deploy and manage Kubernetes resources",
  "version": "1.0.0",
  "packages": [
    {
      "registryType": "oci",
      "identifier": "docker.io/yourusername/kubernetes-manager-mcp:1.0.0",
      "transport": {
        "type": "stdio"
      }
    }
  ]
}
```

The format of `identifier` is `registry/namespace/repository:tag`. For example, `docker.io/user/app:1.0.0` or `ghcr.io/user/app:1.0.0`. The tag can also be specified as a digest.

### Ownership Verification

The MCP Registry verifies ownership of Docker/OCI images by checking for an `io.modelcontextprotocol.server.name` annotation. The value of the `io.modelcontextprotocol.server.name` annotation **MUST** match the server name from `server.json`. For example:

```dockerfile Dockerfile 
LABEL io.modelcontextprotocol.server.name="io.github.username/kubernetes-manager-mcp"
```

## MCPB Packages

For MCPB packages, the MCP Registry currently supports MCPB artifacts hosted via GitHub or GitLab releases.

MCPB packages use `"registryType": "mcpb"` in `server.json`. For example:

```json server.json highlight={9} 
{
  "$schema": "https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json",
  "name": "io.github.username/image-processor-mcp",
  "title": "Image Processor",
  "description": "Process and transform images with various filters",
  "version": "1.0.0",
  "packages": [
    {
      "registryType": "mcpb",
      "identifier": "https://github.com/username/image-processor-mcp/releases/download/v1.0.0/image-processor.mcpb",
      "fileSha256": "fe333e598595000ae021bd27117db32ec69af6987f507ba7a63c90638ff633ce",
      "transport": {
        "type": "stdio"
      }
    }
  ]
}
```

### Verification

The MCPB package URL (`identifier` in `server.json`) **MUST** contain the string "mcp". That can be as part of the `.mcpb` file extension or in the name of the repository.

The package metadata in `server.json` **MUST** include a `fileSha256` property with a SHA-256 hash of the MCPB artifact, which can be computed using the `openssl` command:

```bash 
openssl dgst -sha256 image-processor.mcpb
```

The MCP Registry does not validate this hash; however, MCP clients **do** validate the hash before installation to ensure file integrity. Downstream registries may also implement their own validation.

# Quickstart: Publish an MCP Server to the MCP Registry
Source: https://modelcontextprotocol.io/registry/quickstart

  The MCP Registry is currently in preview. Breaking changes or data resets may occur before general availability. If you encounter any issues, please report them on [GitHub](https://github.com/modelcontextprotocol/registry/issues).

This tutorial will show you how to publish an MCP server written in TypeScript to the MCP Registry using the official `mcp-publisher` CLI tool.

## Prerequisites

* **Node.js** — This tutorial assumes the MCP server is written in TypeScript.
* **npm account** — The MCP Registry only hosts metadata, not artifacts. Before publishing to the MCP Registry, we will publish the MCP server's package to npm, so you will need an [npm](https://www.npmjs.com) account.
* **GitHub account** — The MCP Registry supports [multiple authentication methods](./authentication). For simplicity, this tutorial will use GitHub-based authentication, so you will need a [GitHub](https://github.com/) account.

If you do not have an MCP server written in TypeScript, you can copy the `weather-server-typescript` server from the [`modelcontextprotocol/quickstart-resources` repository](https://github.com/modelcontextprotocol/quickstart-resources) to follow along with this tutorial:

```bash 
git clone --depth 1 git@github.com:modelcontextprotocol/quickstart-resources.git
cp -r quickstart-resources/weather-server-typescript .
rm -rf quickstart-resources
cd weather-server-typescript
```

And edit `package.json` to reflect your information:

```diff package.json 
 {
-  "name": "mcp-quickstart-ts",
-  "version": "1.0.0",
+  "name": "@my-username/mcp-weather-server",
+  "version": "1.0.1",
   "main": "index.js",
```

```diff package.json 
   "license": "ISC",
-  "description": "",
+  "repository": {
+    "type": "git",
+    "url": "https://github.com/my-username/mcp-weather-server.git"
+  },
+  "description": "An MCP server for weather information.",
   "devDependencies": {
```

## Step 1: Add verification information to the package

The MCP Registry verifies that a server's underlying package matches its metadata. For npm packages, this requires adding an `mcpName` property to `package.json`:

```diff package.json 
 {
   "name": "@my-username/mcp-weather-server",
   "version": "1.0.1",
+  "mcpName": "io.github.my-username/weather",
   "main": "index.js",
```

The value of `mcpName` will be your server's name in the MCP Registry.

Because we will be using GitHub-based authentication, `mcpName` **must** start with `io.github.my-username/`.

## Step 2: Publish the package

The MCP Registry only hosts metadata, not artifacts, so we must publish the package to npm before publishing the server to the MCP Registry.

Ensure the distribution files are built:

```bash 
# Navigate to project directory
cd weather-server-typescript

# Install dependencies
npm install

# Build the distribution files
npm run build
```

Then follow npm's [publishing guide](https://docs.npmjs.com/creating-and-publishing-scoped-public-packages). In particular, you will probably need to run the following commands:

```bash 
# If necessary, authenticate to npm
npm adduser

# Publish the package
npm publish --access public
```

You can verify your package is published by visiting its npm URL, such as [https://www.npmjs.com/package/@my-username/mcp-weather-server](https://www.npmjs.com/package/@my-username/mcp-weather-server).

## Step 3: Install `mcp-publisher`

Install the `mcp-publisher` CLI tool using a pre-built binary or [Homebrew](https://brew.sh):

  ```bash macOS/Linux 
  curl -L "https://github.com/modelcontextprotocol/registry/releases/latest/download/mcp-publisher_$(uname -s | tr '[:upper:]' '[:lower:]')_$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/').tar.gz" | tar xz mcp-publisher && sudo mv mcp-publisher /usr/local/bin/
  ```

  ```powershell Windows 
  $arch = if ([System.Runtime.InteropServices.RuntimeInformation]::ProcessArchitecture -eq "Arm64") { "arm64" } else { "amd64" }; Invoke-WebRequest -Uri "https://github.com/modelcontextprotocol/registry/releases/latest/download/mcp-publisher_windows_$arch.tar.gz" -OutFile "mcp-publisher.tar.gz"; tar xf mcp-publisher.tar.gz mcp-publisher.exe; rm mcp-publisher.tar.gz
  # Move mcp-publisher.exe to a directory in your PATH
  ```

  ```bash 
  brew install mcp-publisher
  ```

Verify that `mcp-publisher` is correctly installed by running:

```bash 
mcp-publisher --help
```

You should see output like:

```text Output 
MCP Registry Publisher Tool

Usage:
  mcp-publisher <command> [arguments]

Commands:
  init          Create a server.json file template
  login         Authenticate with the registry
  logout        Clear saved authentication
  publish       Publish server.json to the registry
```

## Step 4: Create `server.json`

The `mcp-publisher init` command can generate a `server.json` template file with some information derived from your project.

In your server project directory, run `mcp-publisher init`:

```bash 
mcp-publisher init
```

Open the generated `server.json` file, and you should see contents like:

```json server.json 
{
  "$schema": "https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json",
  "name": "io.github.my-username/weather",
  "description": "An MCP server for weather information.",
  "repository": {
    "url": "https://github.com/my-username/mcp-weather-server",
    "source": "github"
  },
  "version": "1.0.0",
  "packages": [
    {
      "registryType": "npm",
      "identifier": "@my-username/mcp-weather-server",
      "version": "1.0.0",
      "transport": {
        "type": "stdio"
      },
      "environmentVariables": [
        {
          "description": "Your API key for the service",
          "isRequired": true,
          "format": "string",
          "isSecret": true,
          "name": "YOUR_API_KEY"
        }
      ]
    }
  ]
}
```

Edit the contents as necessary:

```diff server.json 
 {
   "$schema": "https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json",
   "name": "io.github.my-username/weather",
   "description": "An MCP server for weather information.",
   "repository": {
     "url": "https://github.com/my-username/mcp-weather-server",
     "source": "github"
   },
-  "version": "1.0.0",
+  "version": "1.0.1",
   "packages": [
     {
       "registryType": "npm",
       "identifier": "@my-username/mcp-weather-server",
-      "version": "1.0.0",
+      "version": "1.0.1",
       "transport": {
         "type": "stdio"
-      },
-      "environmentVariables": [
-        {
-          "description": "Your API key for the service",
-          "isRequired": true,
-          "format": "string",
-          "isSecret": true,
-          "name": "YOUR_API_KEY"
-        }
-      ]
+      }
     }
   ]
 }
```

The `name` property in `server.json` **must** match the `mcpName` property in `package.json`.

## Step 5: Authenticate with the MCP Registry

For this tutorial, we will authenticate with the MCP Registry using GitHub-based authentication.

Run the `mcp-publisher login` command to initiate authentication:

```bash 
mcp-publisher login github
```

You should see output like:

```text Output 
Logging in with github...

To authenticate, please:
1. Go to: https://github.com/login/device
2. Enter code: ABCD-1234
3. Authorize this application
Waiting for authorization...
```

Visit the link, follow the prompts, and enter the authorization code that was printed in the terminal (e.g., `ABCD-1234` in the above output). Once complete, go back to the terminal, and you should see output like:

```text Output 
Successfully authenticated!
✓ Successfully logged in
```

## Step 6: Publish to the MCP Registry

Finally, publish your server to the MCP Registry using the `mcp-publisher publish` command:

```bash 
mcp-publisher publish
```

You should see output like:

```text Output 
Publishing to https://registry.modelcontextprotocol.io...
✓ Successfully published
✓ Server io.github.my-username/weather version 1.0.1
```

You can verify that your server is published by searching for it using the MCP Registry API:

```bash 
curl "https://registry.modelcontextprotocol.io/v0.1/servers?search=io.github.my-username/weather"
```

You should see your server's metadata in the search results JSON:

```text Output 
{"servers":[{ ... "name":"io.github.my-username/weather" ... }]}
```

## Troubleshooting

| Error Message                                       | Action                                                                                                                                                  |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| "Registry validation failed for package"            | Ensure your package includes the required validation information (e.g, `mcpName` property in `package.json`).                                           |
| "Invalid or expired Registry JWT token"             | Re-authenticate by running `mcp-publisher login github`.                                                                                                |
| "You do not have permission to publish this server" | Your authentication method doesn't match your server's namespace format. With GitHub auth, your server name must start with `io.github.your-username/`. |

## Next Steps

* Learn about [support for other package types](./package-types).
* Learn about [support for remote servers](./remote-servers).
* Learn how to [use other authentication methods](./authentication), such as [DNS authentication](./authentication#dns-authentication) which enables custom domains for server name prefixes.
* Learn how to [automate publishing with GitHub Actions](./github-actions).

# MCP Registry Aggregators
Source: https://modelcontextprotocol.io/registry/registry-aggregators

  The MCP Registry is currently in preview. Breaking changes or data resets may occur before general availability. If you encounter any issues, please report them on [GitHub](https://github.com/modelcontextprotocol/registry/issues).

Aggregators are downstream consumers of the MCP Registry that provide additional value. For example, a server marketplace that provides user ratings and security scanning.

The MCP Registry provides an unauthenticated read-only REST API that aggregators can use to populate their data stores. Aggregators are expected to scrape data on a regular but infrequent basis (e.g., once per hour), and persist the data in their own data store. The MCP Registry **does not provide uptime or data durability guarantees**.

## Consuming the MCP Registry REST API

The base URL for the MCP Registry REST API is `https://registry.modelcontextprotocol.io`. It supports the following endpoints:

* [`GET /v0.1/servers`](https://registry.modelcontextprotocol.io/docs#/operations/list-servers-v0.1) — List all servers.
* [`GET /v0.1/servers/{serverName}/versions`](https://registry.modelcontextprotocol.io/docs#/operations/get-server-versions-v0.1) — List all versions of a server.
* [`GET /v0.1/servers/{serverName}/versions/{version}`](https://registry.modelcontextprotocol.io/docs#/operations/get-server-version-v0.1) — Get a specific version of a server. Use the special version `latest` to get the latest version of the server.

  URL path parameters such as `serverName` and `version` **must** be URL-encoded. For example, `io.modelcontextprotocol/everything` must be encoded as `io.modelcontextprotocol%2Feverything`.

Aggregators will most likely scrape the `GET /v0.1/servers` endpoint.

### Pagination

The `GET /v0.1/servers` endpoint supports cursor-based pagination.

For example, the first page can be fetched using a `limit` query parameter:

```bash 
curl "https://registry.modelcontextprotocol.io/v0.1/servers?limit=100"
```

```jsonc Output highlight={5} 
{
  "servers": [
    /* ... */
  ],
  "metadata": {
    "count": 100,
    "nextCursor": "com.example/my-server:1.0.0",
  },
}
```

Then subsequent pages can be fetched by passing the `nextCursor` value as the `cursor` query parameter:

```bash 
curl "https://registry.modelcontextprotocol.io/v0.1/servers?limit=100&cursor=com.example/my-server:1.0.0"
```

### Filtering Since

The `GET /v0.1/servers` endpoint supports filtering servers that have been updated since a given timestamp.

For example, servers that have been updated since 2025-10-23 can be fetched using an `updated_since` query parameter in [RFC 3339](https://datatracker.ietf.org/doc/html/rfc3339) date-time format:

```bash 
curl "https://registry.modelcontextprotocol.io/v0.1/servers?updated_since=2025-10-23T00:00:00.000Z"
```

## Server Status

Server metadata is generally immutable, except for the `status` field which may be updated to, e.g., `"deprecated"` or `"deleted"`. We recommend that aggregators keep their copy of each server's `status` up to date.

The `"deleted"` status typically indicates that a server has violated our permissive [moderation policy](./moderation-policy), suggesting the server might be spam, malware, or illegal. Aggregators may prefer to remove these servers from their index.

## Acting as a Subregistry

A subregistry is an aggregator that also implements the [OpenAPI spec](https://github.com/modelcontextprotocol/registry/blob/main/docs/reference/api/openapi.yaml) defined by the MCP Registry. This allows clients, such as MCP host applications, to consume server metadata via a standardized interface.

The subregistry OpenAPI spec allows subregistries to inject custom metadata via the `_meta` field. For example, a subregistry could inject user ratings, download counts, and security scan results:

```json server.json highlight={17-26} 
{
  "$schema": "https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json",
  "name": "io.github.username/email-integration-mcp",
  "title": "Email Integration",
  "description": "Send emails and manage email accounts",
  "version": "1.0.0",
  "packages": [
    {
      "registryType": "npm",
      "identifier": "@username/email-integration-mcp",
      "version": "1.0.0",
      "transport": {
        "type": "stdio"
      }
    }
  ],
  "_meta": {
    "com.example.subregistry/custom": {
      "user_rating": 4.5,
      "download_count": 12345,
      "security_scan": {
        "last_scanned": "2025-10-23T12:00:00Z",
        "vulnerabilities_found": 0
      }
    }
  }
}
```

We recommend that custom metadata be put under a key that reflects the subregistry (e.g., `"com.example.subregistry/custom"` in the above example).

# Publishing Remote Servers
Source: https://modelcontextprotocol.io/registry/remote-servers

  The MCP Registry is currently in preview. Breaking changes or data resets may occur before general availability. If you encounter any issues, please report them on [GitHub](https://github.com/modelcontextprotocol/registry/issues).

The MCP Registry supports remote MCP servers via the `remotes` property in `server.json`:

```json server.json highlight={7-12} 
{
  "$schema": "https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json",
  "name": "com.example/acme-analytics",
  "title": "ACME Analytics",
  "description": "Real-time business intelligence and reporting platform",
  "version": "2.0.0",
  "remotes": [
    {
      "type": "streamable-http",
      "url": "https://analytics.example.com/mcp"
    }
  ]
}
```

A remote server **MUST** be publicly accessible at its specified URL.

## Transport Type

Remote servers can use the Streamable HTTP transport (recommended) or the SSE transport. Remote servers can also support both transports simultaneously at different URLs.

Specify the transport by setting the `type` property of the `remotes` entry to either `"streamable-http"` or `"sse"`:

```json server.json highlight={9,13} 
{
  "$schema": "https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json",
  "name": "com.example/acme-analytics",
  "title": "ACME Analytics",
  "description": "Real-time business intelligence and reporting platform",
  "version": "2.0.0",
  "remotes": [
    {
      "type": "streamable-http",
      "url": "https://analytics.example.com/mcp"
    },
    {
      "type": "sse",
      "url": "https://analytics.example.com/sse"
    }
  ]
}
```

## URL Template Variables

Remote servers can define URL template variables using `{curly_braces}` notation. This enables multi-tenant deployments where a single server definition can support multiple endpoints with configurable values:

```json server.json highlight={10-17} 
{
  "$schema": "https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json",
  "name": "com.example/acme-analytics",
  "title": "ACME Analytics",
  "description": "Real-time business intelligence and reporting platform",
  "version": "2.0.0",
  "remotes": [
    {
      "type": "streamable-http",
      "url": "https://{tenant_id}.analytics.example.com/mcp",
      "variables": {
        "tenant_id": {
          "description": "Your tenant identifier (e.g., 'us-cell1', 'emea-cell1')",
          "isRequired": true
        }
      }
    }
  ]
}
```

When configuring this server, users provide their `tenant_id` value, and the URL template gets resolved to the appropriate endpoint (e.g., `https://us-cell1.analytics.example.com/mcp`).

Variables support additional properties like `default`, `choices`, and `isSecret`:

```json server.json highlight={12-22} 
{
  "$schema": "https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json",
  "name": "com.example/multi-region-mcp",
  "title": "Multi-Region MCP",
  "description": "MCP server with regional endpoints",
  "version": "1.0.0",
  "remotes": [
    {
      "type": "streamable-http",
      "url": "https://api.example.com/{region}/mcp",
      "variables": {
        "region": {
          "description": "Deployment region",
          "isRequired": true,
          "choices": [
            "us-east-1",
            "eu-west-1",
            "ap-southeast-1"
          ],
          "default": "us-east-1"
        }
      }
    }
  ]
}
```

## HTTP Headers

MCP clients can be instructed to send specific HTTP headers by adding the `headers` property to the `remotes` entry:

```json server.json highlight={11-18} 
{
  "$schema": "https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json",
  "name": "com.example/acme-analytics",
  "title": "ACME Analytics",
  "description": "Real-time business intelligence and reporting platform",
  "version": "2.0.0",
  "remotes": [
    {
      "type": "streamable-http",
      "url": "https://analytics.example.com/mcp",
      "headers": [
        {
          "name": "X-API-Key",
          "description": "API key for authentication",
          "isRequired": true,
          "isSecret": true
        }
      ]
    }
  ]
}
```

## Supporting Remote and Non-remote Installation

The `remotes` property can coexist with the `packages` property in `server.json` in order to allow MCP host applications to choose the preferred method of installation.

```json server.json highlight={7-22} 
{
  "$schema": "https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json",
  "name": "io.github.username/email-integration-mcp",
  "title": "Email Integration",
  "description": "Send emails and manage email accounts",
  "version": "1.0.0",
  "remotes": [
    {
      "type": "streamable-http",
      "url": "https://email.example.com/mcp"
    }
  ],
  "packages": [
    {
      "registryType": "npm",
      "identifier": "@example/email-integration-mcp",
      "version": "1.0.0",
      "transport": {
        "type": "stdio"
      }
    }
  ]
}
```

# Official MCP Registry Terms of Service
Source: https://modelcontextprotocol.io/registry/terms-of-service

  The MCP Registry is currently in preview. Breaking changes or data resets may occur before general availability. If you encounter any issues, please report them on [GitHub](https://github.com/modelcontextprotocol/registry/issues).

**Effective date: 2025-09-02**

## Overview

These terms (“Terms”) govern your access to and use of the official MCP Registry (the service hosted at [https://registry.modelcontextprotocol.io/](https://registry.modelcontextprotocol.io/) or a successor location) (“Registry”), including submissions or publications of MCP servers, references to MCP servers or to data about such servers and/or their developers (“Registry Data”), and related conduct. The Registry is intended to be a centralized repository of MCP servers developed by community members to facilitate easy access by AI applications.

These terms are governed by the laws of the State of California.

## For All Users

1. No Warranties. The Registry is provided “as is” with no warranties of any kind. That means we don't guarantee the accuracy, completeness, safety, durability, or availability of the Registry, servers included in the registry, or Registry Data. In short, we’re also not responsible for any MCP servers or Registry Data, and we highly recommend that you evaluate each MCP server and its suitability for your intended use case(s) before deciding whether to use it.

2. Access and Use Requirements. To access or use the Registry, you must:
   1. Be at least 18 years old.
   2. Use the Registry, MCP servers in the Registry, and Registry Data only in ways that are legal under the applicable laws of the United States or other countries including the country in which you are a resident or from which you access and use the Registry, and not be barred from accessing or using the Registry under such laws. You will comply with all applicable law, regulation, and third party rights (including, without limitation, laws regarding the import or export of data or software, privacy, intellectual property, and local laws). You will not use the Registry, MCP servers, or Registry Data to encourage or promote illegal activity or the violation of third party rights or terms of service.
   3. Log in via method(s) approved by the Registry maintainers, which may involve using applications or other software owned by third parties.

3. Entity Use. If you are accessing or using the Registry on behalf of an entity, you represent and warrant that you have authority to bind that entity to these Terms. By accepting these Terms, you are doing so on behalf of that entity (and all references to “you” in these Terms refer to that entity).

4. Account Information. In order to access or use the Registry, you may be required to provide certain information (such as identification or contact details) as part of a registration process or in connection with your access or use of the Registry or MCP servers therein. Any information you give must be accurate and up-to-date, and you agree to inform us promptly of any updates. You understand that your use of the Registry may be monitored to ensure quality and verify your compliance with these Terms.

5. Feedback. You are under no obligation to provide feedback or suggestions. If you provide feedback or suggestions about the Registry or the Model Context Protocol, then we (and those we allow) may use such information without obligation to you.

6. Branding. Only use the term “Official MCP Registry” where it is clear it refers to the Registry, and does not imply affiliation, endorsement, or sponsorship. For example, you can permissibly say “Acme Inc. keeps its data up to date by automatically pulling data from the Official MCP Registry” or “This data comes from the Official MCP Registry,” but cannot say “This is the website for the Official MCP Registry,” “We’re the premier destination to view Official MCP Registry data,” or “We’ve partnered with the Official MCP Registry to provide this data.”

7. Modification. We may modify the Terms or any portion to, for example, reflect changes to the law or changes to the Model Context Protocol. We’ll post notice of modifications to the Terms to this website or a successor location. If you do not agree to the modified Terms, you should discontinue your access to and/or use of the Registry. Your continued access to and/or use of the Registry constitutes your acceptance of any modified Terms.

8. Additional Terms. Depending on your intended use case(s), you must also abide by applicable terms below.

## For MCP Developers

9. Prohibitions. By accessing and using the Registry, including by submitting MCP servers and/or Registry Data, you agree not to:
   1. Share malicious or harmful content, such as malware, even in good faith or for research purposes, or perform any action with the intent of introducing any viruses, worms, defects, Trojan horses, malware, or any items of a destructive nature;
   2. Defame, abuse, harass, stalk, or threaten others;
   3. Interfere with or disrupt the Registry or any associated servers or networks;
   4. Submit data with the intent of confusing or misleading others, including but not limited to via spam, posting off-topic marketing content, posting MCP servers in a way that falsely implies affiliation with or endorsement by a third party, or repeatedly posting the same or similar MCP servers under different names;
   5. Promote or facilitate unlawful online gambling or disruptive commercial messages or advertisements;
   6. Use the Registry for any activities where the use or failure of the Registry could lead to death, personal injury, or environmental damage;
   7. Use the Registry to process or store any data that is subject to the International Traffic in Arms Regulations maintained by the U.S. Department of State.

10. License. You agree that metadata about MCP servers you submit (e.g., schema name and description, URLs, identifiers) and other Registry Data is intended to be public, and will be dedicated to the public domain under [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/). By submitting such data, you agree that you have the legal right to make this dedication (i.e., you own the copyright to these submissions or have permission from the copyright owner(s) to do so) and intend to do so. You understand that this dedication is perpetual, irrevocable, and worldwide, and you waive any moral rights you may have in your contributions to the fullest extent permitted by law. This dedication applies only to Registry Data and not to packages in third party registries that you might point to.

11. Privacy and Publicity. You understand that any MCP server metadata you publish may be made public. This includes personal data such as your GitHub username, domain name, or details from your server description. Moreover, you understand that others may process personal information included in your MCP server metadata. For example, subregistries might enrich this data by adding how many stars your GitHub repository has, or perform automated security scanning on your code. By publishing a server, you agree that others may engage in this sort of processing, and you waive rights you might have in some jurisdictions to access, rectify, erase, restrict, or object to such processing.

# Versioning Published MCP Servers
Source: https://modelcontextprotocol.io/registry/versioning

  The MCP Registry is currently in preview. Breaking changes or data resets may occur before general availability. If you encounter any issues, please report them on [GitHub](https://github.com/modelcontextprotocol/registry/issues).

MCP servers **MUST** define a version string in `server.json`. For example:

```json server.json highlight={6} 
{
  "$schema": "https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json",
  "name": "io.github.username/email-integration-mcp",
  "title": "Email Integration",
  "description": "Send emails and manage email accounts",
  "version": "1.0.0",
  "packages": [
    {
      "registryType": "npm",
      "identifier": "@username/email-integration-mcp",
      "version": "1.0.0",
      "transport": {
        "type": "stdio"
      }
    }
  ]
}
```

The version string **MUST** be unique for each publication of the server. Once published, the version string (and other metadata) cannot be changed.

## Version Format

The MCP Registry recommends [semantic versioning](https://semver.org/), but supports any version string format. When a server is published, the MCP Registry will attempt to parse its version as a semantic version string for sorting purposes, and will mark the version as "latest" if appropriate. If parsing fails, the version will always be marked as "latest".

  If a server uses semantic version strings but publishes a new version that does *not* conform to semantic versioning, the new version will be marked as "latest" even if it would otherwise be sorted before the semantic version strings.

As an error prevention mechanism, the MCP Registry prohibits version strings that appear to refer to ranges of versions.

| Example        | Type                | Guidance                       |
| -------------- | ------------------- | ------------------------------ |
| `1.0.0`        | semantic version    | **Recommended**                |
| `2.1.3-alpha`  | semantic prerelease | **Recommended**                |
| `1.0.0-beta.1` | semantic prerelease | **Recommended**                |
| `3.0.0-rc.2`   | semantic prerelease | **Recommended**                |
| `2025.11.25`   | semantic date       | Recommended                    |
| `2025.6.18`    | semantic date       | Recommended **(⚠️Caution!⚠️)** |
| `2025.06.18`   | non-semantic date   | Allowed **(⚠️Caution!⚠️)**     |
| `2025-06-18`   | non-semantic date   | Allowed                        |
| `v1.0`         | prefixed version    | Allowed                        |
| `^1.2.3`       | version range       | Prohibited                     |
| `~1.2.3`       | version range       | Prohibited                     |
| `>=1.2.3`      | version range       | Prohibited                     |
| `<=1.2.3`      | version range       | Prohibited                     |
| `>1.2.3`       | version range       | Prohibited                     |
| `<1.2.3`       | version range       | Prohibited                     |
| `1.x`          | version range       | Prohibited                     |
| `1.2.*`        | version range       | Prohibited                     |
| `1 - 2`        | version range       | Prohibited                     |
| `1.2 \|\| 1.3` | version range       | Prohibited                     |

## Best Practices

### Use Semantic Versioning

Use [semantic versioning](https://semver.org/) for version strings.

### Align Server Version with Package Version

For local servers, align the server version with the underlying package version in order to prevent confusion:

```json server.json highlight={2,7} 
{
  "version": "1.2.3",
  "packages": [
    {
      "registryType": "npm",
      "identifier": "@my-username/my-server",
      "version": "1.2.3",
      "transport": {
        "type": "stdio"
      }
    }
  ]
}
```

If there are multiple underlying packages, use the server version to indicate the overall release version:

```json server.json highlight={2,7,15} 
{
  "version": "1.3.0",
  "packages": [
    {
      "registryType": "npm",
      "identifier": "@my-username/my-server",
      "version": "1.3.0",
      "transport": {
        "type": "stdio"
      }
    },
    {
      "registryType": "nuget",
      "identifier": "MyUsername.MyServer",
      "version": "1.0.0",
      "transport": {
        "type": "stdio"
      }
    }
  ]
}
```

### Align Server Version with Remote API Version

For remote servers with an API version, the server version should align with the API version:

```json server.json highlight={2,6} 
{
  "version": "2.1.0",
  "remotes": [
    {
      "type": "streamable-http",
      "url": "https://api.myservice.com/mcp/v2.1"
    }
  ]
}
```

### Use Prerelease Versions for Registry-only Updates

If you anticipate publishing a server multiple times *without* changing the underlying package or remote URL — for example, to update other parts of the metadata — use semantic prerelease versions:

```json server.json highlight={2} 
{
  "version": "1.2.3-1",
  "packages": [
    {
      "registryType": "npm",
      "identifier": "@my-username/my-server",
      "version": "1.2.3",
      "transport": {
        "type": "stdio"
      }
    }
  ]
}
```

  According to semantic versioning, prerelease versions such as `1.2.3-1` are sorted before regular semantic versions such as `1.2.3`. Therefore, if you publish a prerelease version *after* its corresponding regular version, the prerelease version will **not** be marked as "latest".

## Aggregator Recommendations

MCP Registry aggregators **SHOULD**:

1. Attempt to interpret versions as semantic versions when possible
2. Use the following version comparison rules:
   * If one version is marked as "latest", treat it as later
   * If both versions are valid semantic versions, use semantic versioning comparison rules
   * If neither versions are valid semantic versions, compare published timestamp
   * If one version is a valid semantic version and the other is not, treat the semantic version as later
