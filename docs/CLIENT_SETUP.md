# MCP Client Setup Guide

This guide walks you through setting up an MCP client to interact with Cisco device automation servers.

## Prerequisites

- Python 3.8 or higher (for Python-based client)
- Access to an MCP server
- Network connectivity to the MCP server

## Architecture Overview

```
+----------------+                      +----------------+
|                |                      |                |
|  Application   |                      |   MCP Client   |
|   (Your Code)  |<-------------------->|    Library     |
|                |                      |                |
+----------------+                      +--------+-------+
                                                 |
                                                 | MCP Protocol
                                                 | (HTTP/gRPC)
                                                 |
                                        +--------v-------+
                                        |                |
                                        |   MCP Server   |
                                        |                |
                                        +----------------+
```

## Installation

### Python MCP Client

#### Step 1: Install Dependencies

```bash
# Create a virtual environment
python3 -m venv mcp-client-env
source mcp-client-env/bin/activate  # On Windows: mcp-client-env\Scripts\activate

# Install MCP client library
pip install mcp
pip install asyncio
```

### Node.js MCP Client

```bash
# Initialize npm project
npm init -y

# Install MCP SDK
npm install @modelcontextprotocol/sdk
```

## Python Client Implementation

### Basic Client Setup

Create a file named `cisco_mcp_client.py`:

```python
import asyncio
import json
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
from typing import Optional

class CiscoMCPClient:
    """MCP Client for Cisco device automation"""
    
    def __init__(self, server_script_path: str):
        """
        Initialize the MCP client
        
        Args:
            server_script_path: Path to the MCP server script
        """
        self.server_params = StdioServerParameters(
            command="python",
            args=[server_script_path]
        )
        self.session: Optional[ClientSession] = None
    
    async def connect(self):
        """Establish connection to MCP server"""
        self.read, self.write = await stdio_client(self.server_params).__aenter__()
        self.session = await ClientSession(self.read, self.write).__aenter__()
        await self.session.initialize()
        print("Connected to MCP server")
    
    async def disconnect(self):
        """Close connection to MCP server"""
        if self.session:
            await self.session.__aexit__(None, None, None)
        print("Disconnected from MCP server")
    
    async def list_tools(self):
        """List all available tools from the server"""
        if not self.session:
            raise RuntimeError("Not connected to server")
        
        tools = await self.session.list_tools()
        return tools
    
    async def get_device_config(self, device_ip: str, username: str, 
                                password: str, config_type: str = "running"):
        """
        Get device configuration
        
        Args:
            device_ip: IP address of the Cisco device
            username: Device username
            password: Device password
            config_type: Type of config ("running" or "startup")
        
        Returns:
            Device configuration
        """
        if not self.session:
            raise RuntimeError("Not connected to server")
        
        result = await self.session.call_tool(
            "get_device_config",
            {
                "device_ip": device_ip,
                "username": username,
                "password": password,
                "config_type": config_type
            }
        )
        
        return json.loads(result.content[0].text)
    
    async def execute_command(self, device_ip: str, username: str, 
                             password: str, command: str):
        """
        Execute a command on a Cisco device
        
        Args:
            device_ip: IP address of the Cisco device
            username: Device username
            password: Device password
            command: Command to execute
        
        Returns:
            Command output
        """
        if not self.session:
            raise RuntimeError("Not connected to server")
        
        result = await self.session.call_tool(
            "execute_command",
            {
                "device_ip": device_ip,
                "username": username,
                "password": password,
                "command": command
            }
        )
        
        return json.loads(result.content[0].text)
    
    async def get_interfaces(self, device_ip: str, username: str, password: str):
        """
        Get interface information from device
        
        Args:
            device_ip: IP address of the Cisco device
            username: Device username
            password: Device password
        
        Returns:
            Interface information
        """
        if not self.session:
            raise RuntimeError("Not connected to server")
        
        result = await self.session.call_tool(
            "get_device_interfaces",
            {
                "device_ip": device_ip,
                "username": username,
                "password": password
            }
        )
        
        return json.loads(result.content[0].text)
    
    async def configure_interface(self, device_ip: str, username: str, 
                                  password: str, interface: str, 
                                  config_commands: list):
        """
        Configure an interface on a Cisco device
        
        Args:
            device_ip: IP address of the Cisco device
            username: Device username
            password: Device password
            interface: Interface name
            config_commands: List of configuration commands
        
        Returns:
            Configuration result
        """
        if not self.session:
            raise RuntimeError("Not connected to server")
        
        result = await self.session.call_tool(
            "configure_interface",
            {
                "device_ip": device_ip,
                "username": username,
                "password": password,
                "interface": interface,
                "config_commands": config_commands
            }
        )
        
        return json.loads(result.content[0].text)


async def main():
    """Example usage of the MCP client"""
    
    # Initialize client
    client = CiscoMCPClient("cisco_mcp_server.py")
    
    try:
        # Connect to server
        await client.connect()
        
        # List available tools
        tools = await client.list_tools()
        print("Available tools:")
        for tool in tools:
            print(f"  - {tool.name}: {tool.description}")
        
        # Example: Get device configuration
        config = await client.get_device_config(
            device_ip="192.168.1.1",
            username="admin",
            password="password",
            config_type="running"
        )
        print("\nDevice Configuration:")
        print(json.dumps(config, indent=2))
        
        # Example: Execute command
        result = await client.execute_command(
            device_ip="192.168.1.1",
            username="admin",
            password="password",
            command="show version"
        )
        print("\nCommand Output:")
        print(json.dumps(result, indent=2))
        
        # Example: Get interfaces
        interfaces = await client.get_interfaces(
            device_ip="192.168.1.1",
            username="admin",
            password="password"
        )
        print("\nInterfaces:")
        print(json.dumps(interfaces, indent=2))
        
    finally:
        # Disconnect from server
        await client.disconnect()


if __name__ == "__main__":
    asyncio.run(main())
```

## gRPC Client Implementation

### Step 1: Generate Client Stubs

```bash
# Generate Python gRPC client code (if not already done)
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. cisco_service.proto
```

### Step 2: Implement gRPC Client

Create `cisco_grpc_client.py`:

```python
import grpc
import cisco_service_pb2
import cisco_service_pb2_grpc
import json

class CiscoGRPCClient:
    """gRPC client for Cisco device automation"""
    
    def __init__(self, server_address: str = "localhost:50051"):
        """
        Initialize gRPC client
        
        Args:
            server_address: Address of the gRPC server (host:port)
        """
        self.server_address = server_address
        self.channel = None
        self.stub = None
    
    def connect(self):
        """Establish connection to gRPC server"""
        self.channel = grpc.insecure_channel(self.server_address)
        self.stub = cisco_service_pb2_grpc.CiscoDeviceServiceStub(self.channel)
        print(f"Connected to gRPC server at {self.server_address}")
    
    def disconnect(self):
        """Close connection to gRPC server"""
        if self.channel:
            self.channel.close()
        print("Disconnected from gRPC server")
    
    def get_configuration(self, device_ip: str, username: str, 
                         password: str, config_type: str = "running"):
        """
        Get device configuration
        
        Args:
            device_ip: IP address of the Cisco device
            username: Device username
            password: Device password
            config_type: Type of config ("running" or "startup")
        
        Returns:
            ConfigResponse object
        """
        credentials = cisco_service_pb2.Credentials(
            username=username,
            password=password
        )
        
        request = cisco_service_pb2.ConfigRequest(
            device_ip=device_ip,
            config_type=config_type,
            credentials=credentials
        )
        
        response = self.stub.GetConfiguration(request)
        return response
    
    def execute_command(self, device_ip: str, username: str, 
                       password: str, command: str):
        """
        Execute a command on a Cisco device
        
        Args:
            device_ip: IP address of the Cisco device
            username: Device username
            password: Device password
            command: Command to execute
        
        Returns:
            CommandResponse object
        """
        credentials = cisco_service_pb2.Credentials(
            username=username,
            password=password
        )
        
        request = cisco_service_pb2.CommandRequest(
            device_ip=device_ip,
            command=command,
            credentials=credentials
        )
        
        response = self.stub.ExecuteCommand(request)
        return response
    
    def get_interfaces(self, device_ip: str, username: str, password: str):
        """
        Get interface information
        
        Args:
            device_ip: IP address of the Cisco device
            username: Device username
            password: Device password
        
        Returns:
            InterfaceResponse object
        """
        credentials = cisco_service_pb2.Credentials(
            username=username,
            password=password
        )
        
        request = cisco_service_pb2.InterfaceRequest(
            device_ip=device_ip,
            credentials=credentials
        )
        
        response = self.stub.GetInterfaces(request)
        return response
    
    def configure_interface(self, device_ip: str, username: str, 
                          password: str, interface_name: str, 
                          config_commands: list):
        """
        Configure an interface
        
        Args:
            device_ip: IP address of the Cisco device
            username: Device username
            password: Device password
            interface_name: Name of the interface
            config_commands: List of configuration commands
        
        Returns:
            InterfaceConfigResponse object
        """
        credentials = cisco_service_pb2.Credentials(
            username=username,
            password=password
        )
        
        request = cisco_service_pb2.InterfaceConfigRequest(
            device_ip=device_ip,
            interface_name=interface_name,
            config_commands=config_commands,
            credentials=credentials
        )
        
        response = self.stub.ConfigureInterface(request)
        return response


def main():
    """Example usage of gRPC client"""
    
    # Initialize and connect
    client = CiscoGRPCClient("localhost:50051")
    client.connect()
    
    try:
        # Get device configuration
        config_response = client.get_configuration(
            device_ip="192.168.1.1",
            username="admin",
            password="password",
            config_type="running"
        )
        print(f"Configuration retrieved: {config_response.status_code}")
        print(f"Message: {config_response.message}")
        
        # Execute command
        cmd_response = client.execute_command(
            device_ip="192.168.1.1",
            username="admin",
            password="password",
            command="show version"
        )
        print(f"\nCommand executed: {cmd_response.status_code}")
        print(f"Output:\n{cmd_response.output}")
        
        # Get interfaces
        int_response = client.get_interfaces(
            device_ip="192.168.1.1",
            username="admin",
            password="password"
        )
        print(f"\nInterfaces retrieved: {int_response.status_code}")
        for interface in int_response.interfaces:
            print(f"  - {interface.name}: {interface.status}")
        
    finally:
        client.disconnect()


if __name__ == "__main__":
    main()
```

## Node.js Client Implementation

Create `ciscoMCPClient.js`:

```javascript
const { Client } = require("@modelcontextprotocol/sdk/client/index.js");
const { StdioClientTransport } = require("@modelcontextprotocol/sdk/client/stdio.js");

class CiscoMCPClient {
  constructor(serverScriptPath) {
    this.serverScriptPath = serverScriptPath;
    this.client = null;
  }

  async connect() {
    const transport = new StdioClientTransport({
      command: "python",
      args: [this.serverScriptPath],
    });

    this.client = new Client(
      {
        name: "cisco-automation-client",
        version: "1.0.0",
      },
      {
        capabilities: {},
      }
    );

    await this.client.connect(transport);
    console.log("Connected to MCP server");
  }

  async disconnect() {
    if (this.client) {
      await this.client.close();
      console.log("Disconnected from MCP server");
    }
  }

  async listTools() {
    if (!this.client) {
      throw new Error("Not connected to server");
    }

    const result = await this.client.listTools();
    return result.tools;
  }

  async getDeviceConfig(deviceIp, username, password, configType = "running") {
    if (!this.client) {
      throw new Error("Not connected to server");
    }

    const result = await this.client.callTool({
      name: "get_device_config",
      arguments: {
        device_ip: deviceIp,
        username: username,
        password: password,
        config_type: configType,
      },
    });

    return JSON.parse(result.content[0].text);
  }

  async executeCommand(deviceIp, username, password, command) {
    if (!this.client) {
      throw new Error("Not connected to server");
    }

    const result = await this.client.callTool({
      name: "execute_command",
      arguments: {
        device_ip: deviceIp,
        username: username,
        password: password,
        command: command,
      },
    });

    return JSON.parse(result.content[0].text);
  }

  async getInterfaces(deviceIp, username, password) {
    if (!this.client) {
      throw new Error("Not connected to server");
    }

    const result = await this.client.callTool({
      name: "get_device_interfaces",
      arguments: {
        device_ip: deviceIp,
        username: username,
        password: password,
      },
    });

    return JSON.parse(result.content[0].text);
  }

  async configureInterface(deviceIp, username, password, interfaceName, configCommands) {
    if (!this.client) {
      throw new Error("Not connected to server");
    }

    const result = await this.client.callTool({
      name: "configure_interface",
      arguments: {
        device_ip: deviceIp,
        username: username,
        password: password,
        interface: interfaceName,
        config_commands: configCommands,
      },
    });

    return JSON.parse(result.content[0].text);
  }
}

// Example usage
async function main() {
  const client = new CiscoMCPClient("cisco_mcp_server.py");

  try {
    await client.connect();

    // List tools
    const tools = await client.listTools();
    console.log("Available tools:");
    tools.forEach((tool) => {
      console.log(`  - ${tool.name}: ${tool.description}`);
    });

    // Get device configuration
    const config = await client.getDeviceConfig(
      "192.168.1.1",
      "admin",
      "password",
      "running"
    );
    console.log("\nDevice Configuration:");
    console.log(JSON.stringify(config, null, 2));

    // Execute command
    const result = await client.executeCommand(
      "192.168.1.1",
      "admin",
      "password",
      "show version"
    );
    console.log("\nCommand Output:");
    console.log(JSON.stringify(result, null, 2));

  } finally {
    await client.disconnect();
  }
}

if (require.main === module) {
  main().catch(console.error);
}

module.exports = CiscoMCPClient;
```

## Error Handling

### Python Error Handling

```python
import asyncio
from mcp.types import McpError

async def safe_execute():
    client = CiscoMCPClient("cisco_mcp_server.py")
    
    try:
        await client.connect()
        
        result = await client.execute_command(
            device_ip="192.168.1.1",
            username="admin",
            password="password",
            command="show version"
        )
        
        if result.get("status") == "error":
            print(f"Error: {result.get('message')}")
        else:
            print(f"Success: {result.get('output')}")
            
    except McpError as e:
        print(f"MCP Error: {e}")
    except ConnectionError as e:
        print(f"Connection Error: {e}")
    except Exception as e:
        print(f"Unexpected Error: {e}")
    finally:
        await client.disconnect()
```

### Retry Logic

```python
import time

async def execute_with_retry(client, max_retries=3, delay=1):
    """Execute command with retry logic"""
    for attempt in range(max_retries):
        try:
            result = await client.execute_command(
                device_ip="192.168.1.1",
                username="admin",
                password="password",
                command="show version"
            )
            return result
        except Exception as e:
            if attempt < max_retries - 1:
                print(f"Attempt {attempt + 1} failed: {e}")
                await asyncio.sleep(delay * (2 ** attempt))  # Exponential backoff
            else:
                raise
```

## Testing the Client

### Unit Tests

Create `test_client.py`:

```python
import unittest
import asyncio
from unittest.mock import Mock, patch
from cisco_mcp_client import CiscoMCPClient

class TestCiscoMCPClient(unittest.TestCase):
    
    def setUp(self):
        self.client = CiscoMCPClient("cisco_mcp_server.py")
    
    @patch('cisco_mcp_client.stdio_client')
    async def test_connect(self, mock_stdio):
        """Test client connection"""
        await self.client.connect()
        self.assertIsNotNone(self.client.session)
    
    async def test_get_device_config(self):
        """Test get device configuration"""
        # Mock the session
        self.client.session = Mock()
        self.client.session.call_tool = Mock(return_value=Mock(
            content=[Mock(text='{"status": "success"}')]
        ))
        
        result = await self.client.get_device_config(
            "192.168.1.1", "admin", "password"
        )
        
        self.assertEqual(result["status"], "success")

if __name__ == '__main__':
    unittest.main()
```

### Integration Tests

```python
async def integration_test():
    """Integration test with actual server"""
    client = CiscoMCPClient("cisco_mcp_server.py")
    
    try:
        await client.connect()
        
        # Test listing tools
        tools = await client.list_tools()
        assert len(tools) > 0, "No tools available"
        
        # Test each tool (with mock devices)
        print("All integration tests passed")
        
    finally:
        await client.disconnect()
```

## Best Practices

1. **Connection Management**: Always use try-finally blocks to ensure proper disconnection
2. **Error Handling**: Implement comprehensive error handling for network issues
3. **Credential Security**: Never hardcode credentials; use environment variables or secure vaults
4. **Timeout Configuration**: Set appropriate timeouts for all operations
5. **Logging**: Implement detailed logging for troubleshooting
6. **Retries**: Use exponential backoff for retry logic
7. **Connection Pooling**: Reuse client connections when possible
8. **Resource Cleanup**: Properly close all connections and free resources

## Configuration

### Environment Variables

```bash
export MCP_SERVER_PATH="./cisco_mcp_server.py"
export CISCO_USERNAME="admin"
export CISCO_PASSWORD="secure_password"
export MCP_TIMEOUT="30"
```

### Configuration File

Create `client_config.json`:

```json
{
  "server": {
    "path": "./cisco_mcp_server.py",
    "transport": "stdio"
  },
  "defaults": {
    "timeout": 30,
    "retry_attempts": 3,
    "retry_delay": 1
  },
  "logging": {
    "level": "INFO",
    "file": "mcp_client.log"
  }
}
```

## Troubleshooting

### Common Issues

1. **Connection Refused**: Verify server is running and accessible
2. **Timeout Errors**: Increase timeout values or check network connectivity
3. **Authentication Errors**: Verify credentials are correct
4. **Tool Not Found**: Ensure server supports the requested tool
5. **Schema Validation Errors**: Check that request parameters match the schema

### Debug Mode

Enable verbose logging:

```python
import logging

logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
```

## Next Steps

- [Examples](./EXAMPLES.md) - See practical examples and use cases
- [API Reference](./API_REFERENCE.md) - Detailed API documentation
- [Server Setup](./SERVER_SETUP.md) - Learn about server implementation
