# MCP Server Setup Guide

This guide walks you through setting up an MCP server for Cisco device automation.

## Prerequisites

- Python 3.8 or higher (for Python-based server)
- Network access to Cisco devices
- Device credentials with appropriate permissions
- Basic understanding of network protocols (SSH, NETCONF, RESTCONF)

## Architecture Overview

```
+----------------+     MCP Protocol      +----------------+
|                |<-------------------->|                |
|  MCP Client    |   (HTTP/gRPC)        |   MCP Server   |
|                |                      |                |
+----------------+                      +-------+--------+
                                                |
                                                | SSH/NETCONF
                                                |
                                        +-------v--------+
                                        | Cisco Devices  |
                                        +----------------+
```

## Installation

### Option 1: Python-based MCP Server

#### Step 1: Install Dependencies

```bash
# Create a virtual environment
python3 -m venv mcp-server-env
source mcp-server-env/bin/activate  # On Windows: mcp-server-env\Scripts\activate

# Install required packages
pip install mcp
pip install netmiko  # For SSH connections to Cisco devices
pip install ncclient  # For NETCONF connections
pip install requests  # For REST API calls
```

#### Step 2: Create Server Implementation

Create a file named `cisco_mcp_server.py`:

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent
import netmiko
import json
import logging

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Initialize MCP server
app = Server("cisco-automation-server")

# Device connection pool
device_connections = {}

def get_device_connection(device_ip: str, username: str, password: str):
    """Establish or retrieve existing SSH connection to Cisco device"""
    if device_ip not in device_connections:
        device = {
            'device_type': 'cisco_ios',
            'host': device_ip,
            'username': username,
            'password': password,
            'timeout': 30,
        }
        try:
            connection = netmiko.ConnectHandler(**device)
            device_connections[device_ip] = connection
            logger.info(f"Connected to device {device_ip}")
        except Exception as e:
            logger.error(f"Failed to connect to {device_ip}: {str(e)}")
            raise
    return device_connections[device_ip]

@app.list_tools()
async def list_tools() -> list[Tool]:
    """List available tools for Cisco device automation"""
    return [
        Tool(
            name="get_device_config",
            description="Retrieve running or startup configuration from a Cisco device",
            inputSchema={
                "type": "object",
                "properties": {
                    "device_ip": {
                        "type": "string",
                        "description": "IP address of the Cisco device"
                    },
                    "config_type": {
                        "type": "string",
                        "enum": ["running", "startup"],
                        "description": "Type of configuration to retrieve",
                        "default": "running"
                    },
                    "username": {
                        "type": "string",
                        "description": "Device username"
                    },
                    "password": {
                        "type": "string",
                        "description": "Device password"
                    }
                },
                "required": ["device_ip", "username", "password"]
            }
        ),
        Tool(
            name="execute_command",
            description="Execute a command on a Cisco device",
            inputSchema={
                "type": "object",
                "properties": {
                    "device_ip": {
                        "type": "string",
                        "description": "IP address of the Cisco device"
                    },
                    "command": {
                        "type": "string",
                        "description": "Command to execute"
                    },
                    "username": {
                        "type": "string",
                        "description": "Device username"
                    },
                    "password": {
                        "type": "string",
                        "description": "Device password"
                    }
                },
                "required": ["device_ip", "command", "username", "password"]
            }
        ),
        Tool(
            name="get_device_interfaces",
            description="Get interface status and configuration from a Cisco device",
            inputSchema={
                "type": "object",
                "properties": {
                    "device_ip": {
                        "type": "string",
                        "description": "IP address of the Cisco device"
                    },
                    "username": {
                        "type": "string",
                        "description": "Device username"
                    },
                    "password": {
                        "type": "string",
                        "description": "Device password"
                    }
                },
                "required": ["device_ip", "username", "password"]
            }
        ),
        Tool(
            name="configure_interface",
            description="Configure an interface on a Cisco device",
            inputSchema={
                "type": "object",
                "properties": {
                    "device_ip": {
                        "type": "string",
                        "description": "IP address of the Cisco device"
                    },
                    "interface": {
                        "type": "string",
                        "description": "Interface name (e.g., GigabitEthernet0/1)"
                    },
                    "config_commands": {
                        "type": "array",
                        "items": {"type": "string"},
                        "description": "List of configuration commands"
                    },
                    "username": {
                        "type": "string",
                        "description": "Device username"
                    },
                    "password": {
                        "type": "string",
                        "description": "Device password"
                    }
                },
                "required": ["device_ip", "interface", "config_commands", "username", "password"]
            }
        )
    ]

@app.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    """Handle tool execution requests"""
    
    try:
        if name == "get_device_config":
            connection = get_device_connection(
                arguments["device_ip"],
                arguments["username"],
                arguments["password"]
            )
            
            if arguments.get("config_type", "running") == "running":
                config = connection.send_command("show running-config")
            else:
                config = connection.send_command("show startup-config")
            
            return [TextContent(
                type="text",
                text=json.dumps({
                    "status": "success",
                    "device": arguments["device_ip"],
                    "config_type": arguments.get("config_type", "running"),
                    "configuration": config
                }, indent=2)
            )]
        
        elif name == "execute_command":
            connection = get_device_connection(
                arguments["device_ip"],
                arguments["username"],
                arguments["password"]
            )
            
            output = connection.send_command(arguments["command"])
            
            return [TextContent(
                type="text",
                text=json.dumps({
                    "status": "success",
                    "device": arguments["device_ip"],
                    "command": arguments["command"],
                    "output": output
                }, indent=2)
            )]
        
        elif name == "get_device_interfaces":
            connection = get_device_connection(
                arguments["device_ip"],
                arguments["username"],
                arguments["password"]
            )
            
            output = connection.send_command("show ip interface brief")
            
            return [TextContent(
                type="text",
                text=json.dumps({
                    "status": "success",
                    "device": arguments["device_ip"],
                    "interfaces": output
                }, indent=2)
            )]
        
        elif name == "configure_interface":
            connection = get_device_connection(
                arguments["device_ip"],
                arguments["username"],
                arguments["password"]
            )
            
            # Enter configuration mode
            config_commands = [
                f"interface {arguments['interface']}"
            ] + arguments["config_commands"]
            
            output = connection.send_config_set(config_commands)
            
            return [TextContent(
                type="text",
                text=json.dumps({
                    "status": "success",
                    "device": arguments["device_ip"],
                    "interface": arguments["interface"],
                    "output": output
                }, indent=2)
            )]
        
        else:
            return [TextContent(
                type="text",
                text=json.dumps({
                    "status": "error",
                    "message": f"Unknown tool: {name}"
                }, indent=2)
            )]
    
    except Exception as e:
        logger.error(f"Error executing tool {name}: {str(e)}")
        return [TextContent(
            type="text",
            text=json.dumps({
                "status": "error",
                "message": str(e)
            }, indent=2)
        )]

async def main():
    """Run the MCP server"""
    async with stdio_server() as (read_stream, write_stream):
        await app.run(
            read_stream,
            write_stream,
            app.create_initialization_options()
        )

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

#### Step 3: Configure Server Settings

Create a configuration file `server_config.json`:

```json
{
  "server": {
    "name": "cisco-automation-server",
    "version": "1.0.0",
    "description": "MCP server for Cisco device automation"
  },
  "transport": {
    "type": "stdio"
  },
  "security": {
    "tls_enabled": false,
    "credential_storage": "environment"
  },
  "logging": {
    "level": "INFO",
    "file": "mcp_server.log"
  }
}
```

#### Step 4: Run the Server

```bash
# Run the server
python cisco_mcp_server.py

# The server will start and listen for MCP client connections via stdio
```

### Option 2: gRPC-based MCP Server

#### Step 1: Define Protocol Buffers

Create a file named `cisco_service.proto`:

```protobuf
syntax = "proto3";

package cisco.mcp;

service CiscoDeviceService {
  rpc GetConfiguration(ConfigRequest) returns (ConfigResponse) {}
  rpc ExecuteCommand(CommandRequest) returns (CommandResponse) {}
  rpc GetInterfaces(InterfaceRequest) returns (InterfaceResponse) {}
  rpc ConfigureInterface(InterfaceConfigRequest) returns (InterfaceConfigResponse) {}
  rpc StreamDeviceStatus(StatusRequest) returns (stream StatusUpdate) {}
}

message ConfigRequest {
  string device_ip = 1;
  string config_type = 2;  // "running" or "startup"
  Credentials credentials = 3;
}

message ConfigResponse {
  string configuration = 1;
  int32 status_code = 2;
  string message = 3;
}

message CommandRequest {
  string device_ip = 1;
  string command = 2;
  Credentials credentials = 3;
}

message CommandResponse {
  string output = 1;
  int32 status_code = 2;
  string message = 3;
}

message InterfaceRequest {
  string device_ip = 1;
  Credentials credentials = 2;
}

message InterfaceResponse {
  repeated Interface interfaces = 1;
  int32 status_code = 2;
  string message = 3;
}

message InterfaceConfigRequest {
  string device_ip = 1;
  string interface_name = 2;
  repeated string config_commands = 3;
  Credentials credentials = 4;
}

message InterfaceConfigResponse {
  string output = 1;
  int32 status_code = 2;
  string message = 3;
}

message StatusRequest {
  string device_ip = 1;
  int32 interval_seconds = 2;
  Credentials credentials = 3;
}

message StatusUpdate {
  string device_ip = 1;
  string status = 2;
  int64 timestamp = 3;
  map<string, string> metrics = 4;
}

message Credentials {
  string username = 1;
  string password = 2;
}

message Interface {
  string name = 1;
  string status = 2;
  string protocol = 3;
  string ip_address = 4;
}
```

#### Step 2: Generate gRPC Code

```bash
# Install gRPC tools
pip install grpcio grpcio-tools

# Generate Python code from proto file
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. cisco_service.proto
```

#### Step 3: Implement gRPC Server

Create `cisco_grpc_server.py`:

```python
import grpc
from concurrent import futures
import netmiko
import cisco_service_pb2
import cisco_service_pb2_grpc
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class CiscoDeviceServicer(cisco_service_pb2_grpc.CiscoDeviceServiceServicer):
    
    def __init__(self):
        self.connections = {}
    
    def _get_connection(self, device_ip, username, password):
        """Get or create device connection"""
        if device_ip not in self.connections:
            device = {
                'device_type': 'cisco_ios',
                'host': device_ip,
                'username': username,
                'password': password,
                'timeout': 30,
            }
            self.connections[device_ip] = netmiko.ConnectHandler(**device)
        return self.connections[device_ip]
    
    def GetConfiguration(self, request, context):
        """Retrieve device configuration"""
        try:
            connection = self._get_connection(
                request.device_ip,
                request.credentials.username,
                request.credentials.password
            )
            
            if request.config_type == "startup":
                config = connection.send_command("show startup-config")
            else:
                config = connection.send_command("show running-config")
            
            return cisco_service_pb2.ConfigResponse(
                configuration=config,
                status_code=0,
                message="Success"
            )
        except Exception as e:
            logger.error(f"Error getting configuration: {str(e)}")
            return cisco_service_pb2.ConfigResponse(
                configuration="",
                status_code=1,
                message=str(e)
            )
    
    def ExecuteCommand(self, request, context):
        """Execute command on device"""
        try:
            connection = self._get_connection(
                request.device_ip,
                request.credentials.username,
                request.credentials.password
            )
            
            output = connection.send_command(request.command)
            
            return cisco_service_pb2.CommandResponse(
                output=output,
                status_code=0,
                message="Success"
            )
        except Exception as e:
            logger.error(f"Error executing command: {str(e)}")
            return cisco_service_pb2.CommandResponse(
                output="",
                status_code=1,
                message=str(e)
            )

def serve():
    """Start gRPC server"""
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    cisco_service_pb2_grpc.add_CiscoDeviceServiceServicer_to_server(
        CiscoDeviceServicer(), server
    )
    server.add_insecure_port('[::]:50051')
    server.start()
    logger.info("gRPC server started on port 50051")
    server.wait_for_termination()

if __name__ == '__main__':
    serve()
```

#### Step 4: Run gRPC Server

```bash
python cisco_grpc_server.py
```

## Configuration Management

### Environment Variables

Set up environment variables for sensitive data:

```bash
export CISCO_DEFAULT_USERNAME="admin"
export CISCO_DEFAULT_PASSWORD="secure_password"
export MCP_SERVER_PORT="50051"
export MCP_LOG_LEVEL="INFO"
```

### Configuration File

Create `.env` file:

```bash
CISCO_DEFAULT_USERNAME=admin
CISCO_DEFAULT_PASSWORD=secure_password
MCP_SERVER_PORT=50051
MCP_LOG_LEVEL=INFO
DEVICE_TIMEOUT=30
MAX_CONNECTIONS=10
```

## Security Best Practices

1. **Credential Management**: Use environment variables or secret managers (HashiCorp Vault, AWS Secrets Manager)
2. **TLS/SSL**: Enable TLS for gRPC connections in production
3. **Authentication**: Implement token-based authentication for clients
4. **Rate Limiting**: Limit requests per client to prevent abuse
5. **Logging**: Log all operations for audit trails
6. **Input Validation**: Validate all inputs before processing
7. **Connection Limits**: Set maximum concurrent connections

## Testing the Server

### Using MCP Inspector

```bash
# Install MCP inspector
npm install -g @modelcontextprotocol/inspector

# Run inspector
mcp-inspector python cisco_mcp_server.py
```

### Manual Testing with Python

```python
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def test_server():
    server_params = StdioServerParameters(
        command="python",
        args=["cisco_mcp_server.py"]
    )
    
    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            
            # List available tools
            tools = await session.list_tools()
            print("Available tools:", tools)
            
            # Test get_device_config tool
            result = await session.call_tool(
                "get_device_config",
                {
                    "device_ip": "192.168.1.1",
                    "username": "admin",
                    "password": "password",
                    "config_type": "running"
                }
            )
            print("Result:", result)

if __name__ == "__main__":
    asyncio.run(test_server())
```

## Monitoring and Logging

### Log Configuration

Add comprehensive logging:

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('mcp_server.log'),
        logging.StreamHandler()
    ]
)
```

### Health Check Endpoint

Add health check for gRPC:

```python
def health_check(self, request, context):
    """Health check endpoint"""
    return cisco_service_pb2.HealthResponse(
        status="healthy",
        timestamp=int(time.time())
    )
```

## Troubleshooting

### Common Issues

1. **Connection Timeout**: Increase timeout values in device configuration
2. **Authentication Failures**: Verify credentials and device AAA configuration
3. **Command Failures**: Check device privilege level and command syntax
4. **Memory Issues**: Implement connection pooling with limits
5. **Port Conflicts**: Ensure gRPC port is not in use

### Debug Mode

Enable debug logging:

```python
logging.basicConfig(level=logging.DEBUG)
```

## Next Steps

- [Client Setup Guide](./CLIENT_SETUP.md) - Set up an MCP client
- [Examples](./EXAMPLES.md) - See practical examples
- [API Reference](./API_REFERENCE.md) - Detailed API documentation
