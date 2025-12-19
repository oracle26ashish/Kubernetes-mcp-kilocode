# Kubernetes-mcp-kilocode
> How to setup Kubernetes MCP in kilocode

# Setting Up Kubernetes MCP Server in KiloCode

This guide provides step-by-step instructions to set up a local Kubernetes MCP server in your KiloCode workspace, allowing you to manage your Kubernetes cluster through AI-assisted commands.

## Prerequisites

- Node.js and npm installed
- Access to npm registry (may require IT approval for @modelcontextprotocol/sdk)
- kubectl configured and connected to your Kubernetes cluster
- KiloCode workspace

## Step 1: Create the MCP Server Directory Structure

```bash
mkdir -p mcp/kubernetes-server/src
```

## Step 2: Create package.json

Create `mcp/kubernetes-server/package.json` with the following content:

```json
{
  "name": "kubernetes-server",
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "build": "tsc && chmod +x build/index.js"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^0.4.0",
    "zod": "^3.22.4"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "typescript": "^5.0.0"
  }
}
```

## Step 3: Create tsconfig.json

Create `mcp/kubernetes-server/tsconfig.json` with the following content:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "node",
    "outDir": "./build",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "build"]
}
```

## Step 4: Create the Server Implementation

Create `mcp/kubernetes-server/src/index.ts` with the following content:

```typescript
#!/usr/bin/env node
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";
import { execSync } from "child_process";

// Create an MCP server
const server = new McpServer({
  name: "kubernetes-server",
  version: "0.1.0"
});

// Tool for getting pods
server.tool(
  "get_pods",
  {
    namespace: z.string().optional().describe("Namespace to list pods from"),
    selector: z.string().optional().describe("Label selector"),
    output: z.string().optional().describe("Output format (json, yaml, wide, etc.)")
  },
  async ({ namespace, selector, output = "wide" }) => {
    try {
      let cmd = `kubectl get pods`;
      if (namespace) cmd += ` -n ${namespace}`;
      if (selector) cmd += ` -l ${selector}`;
      cmd += ` -o ${output}`;
      const result = execSync(cmd, { encoding: 'utf8' });
      return {
        content: [{ type: "text", text: result }]
      };
    } catch (error: any) {
      return {
        content: [{ type: "text", text: `Error: ${error.message}` }],
        isError: true
      };
    }
  }
);

// Tool for describing a pod
server.tool(
  "describe_pod",
  {
    name: z.string().describe("Pod name"),
    namespace: z.string().optional().describe("Namespace")
  },
  async ({ name, namespace }) => {
    try {
      let cmd = `kubectl describe pod ${name}`;
      if (namespace) cmd += ` -n ${namespace}`;
      const result = execSync(cmd, { encoding: 'utf8' });
      return {
        content: [{ type: "text", text: result }]
      };
    } catch (error: any) {
      return {
        content: [{ type: "text", text: `Error: ${error.message}` }],
        isError: true
      };
    }
  }
);

// Tool for getting logs
server.tool(
  "logs",
  {
    name: z.string().describe("Pod name"),
    namespace: z.string().optional().describe("Namespace"),
    container: z.string().optional().describe("Container name"),
    tail: z.number().optional().describe("Number of lines to tail")
  },
  async ({ name, namespace, container, tail }) => {
    try {
      let cmd = `kubectl logs ${name}`;
      if (namespace) cmd += ` -n ${namespace}`;
      if (container) cmd += ` -c ${container}`;
      if (tail) cmd += ` --tail=${tail}`;
      const result = execSync(cmd, { encoding: 'utf8' });
      return {
        content: [{ type: "text", text: result }]
      };
    } catch (error: any) {
      return {
        content: [{ type: "text", text: `Error: ${error.message}` }],
        isError: true
      };
    }
  }
);

// Tool for getting services
server.tool(
  "get_services",
  {
    namespace: z.string().optional().describe("Namespace"),
    output: z.string().optional().describe("Output format")
  },
  async ({ namespace, output = "wide" }) => {
    try {
      let cmd = `kubectl get services`;
      if (namespace) cmd += ` -n ${namespace}`;
      cmd += ` -o ${output}`;
      const result = execSync(cmd, { encoding: 'utf8' });
      return {
        content: [{ type: "text", text: result }]
      };
    } catch (error: any) {
      return {
        content: [{ type: "text", text: `Error: ${error.message}` }],
        isError: true
      };
    }
  }
);

// Start the server
const transport = new StdioServerTransport();
await server.connect(transport);
console.error('Kubernetes MCP server running on stdio');
```

## Step 5: Install Dependencies

Navigate to the server directory and install dependencies:

```bash
cd mcp/kubernetes-server
npm install
```

**Note:** If you encounter a 403 error, contact your IT administrator to allow access to the @modelcontextprotocol/sdk package on npmjs.org.

## Step 6: Build the Server

Compile the TypeScript code:

```bash
npm run build
```

## Step 7: Configure KiloCode MCP Settings

Update `.kilocode/mcp.json` in your workspace root to include the local server:

```json
{
  "mcpServers": {
    "kubernetes-mcp-server": {
      "command": "node",
      "args": ["mcp/kubernetes-server/build/index.js"]
    }
  }
}
```

## Step 8: Verify Setup

Restart KiloCode or reload the window to load the new MCP server. You can now use commands like:

- "Get all pods in the advisor namespace"
- "Describe the advisor-service pod"
- "Show logs for a specific pod"
- "List services in the cluster"

## Troubleshooting

- **403 npm error:** Security policy blocking package access
- **Build fails:** Ensure TypeScript and dependencies are installed
- **MCP not loading:** Check the .kilocode/mcp.json syntax and path

## Available Tools

- `get_pods`: List pods with optional namespace and label filters
- `describe_pod`: Get detailed pod information
- `logs`: Retrieve container logs
- `get_services`: List services in namespaces
