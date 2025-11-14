# MCP-Cisco

Comprehensive documentation and examples for automating Cisco devices using the Model Context Protocol (MCP).

## Overview

This repository provides complete documentation, implementation examples, and best practices for using MCP (Model Context Protocol) to automate Cisco network devices. MCP enables seamless integration between AI applications and network infrastructure through a standardized client-server architecture.

## What is MCP?

Model Context Protocol (MCP) is an open protocol that standardizes how applications provide context to Large Language Models (LLMs). It enables:

- Seamless integration between AI systems and network infrastructure
- Schema-driven communication for type safety and validation
- Support for multiple transport protocols (HTTP, gRPC)
- Intent-based networking and state management

## Documentation Structure

### Core Documentation

1. **[MCP Overview](./docs/MCP_OVERVIEW.md)**
   - What are MCP servers and clients
   - How MCP servers and clients exchange schema-driven state and intent
   - Communication protocols (gRPC/HTTP endpoints)
   - Security considerations and best practices

2. **[Server Setup Guide](./docs/SERVER_SETUP.md)**
   - Setting up Python-based MCP servers
   - Implementing gRPC-based servers
   - Configuration management
   - Security and monitoring

3. **[Client Setup Guide](./docs/CLIENT_SETUP.md)**
   - Python MCP client implementation
   - gRPC client setup
   - Node.js client implementation
   - Error handling and best practices

4. **[Practical Examples](./docs/EXAMPLES.md)**
   - Basic device configuration
   - Interface management
   - Multi-device operations
   - Configuration backup and restore
   - Network monitoring
   - VLAN and routing configuration
   - Security configuration (ACLs)

## Quick Start

### Prerequisites

- Python 3.8 or higher
- Network access to Cisco devices
- Basic understanding of network protocols

### Installation

```bash
# Clone the repository
git clone https://github.com/ters-golemi/MCP-Cisco.git
cd MCP-Cisco

# Create virtual environment
python3 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install mcp netmiko ncclient
```

### Basic Usage

```python
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def main():
    # Connect to MCP server
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

if __name__ == "__main__":
    asyncio.run(main())
```

## Key Features

### MCP Servers

- Expose Cisco device capabilities through standardized tools
- Support for SSH, NETCONF, and RESTCONF connections
- Schema-driven API definitions
- Connection pooling and state management
- Comprehensive error handling

### MCP Clients

- Simple API for device automation
- Support for both Python and Node.js
- Async/await patterns for efficient operations
- Built-in retry logic and error handling
- Multi-device orchestration

### Communication Protocols

#### HTTP/REST
- Simple and widely supported
- Easy to debug and test
- Works through standard firewalls
- JSON-based request/response

#### gRPC
- High performance binary protocol
- Bidirectional streaming support
- Strong typing with Protocol Buffers
- Automatic code generation

## Use Cases

- Automated device configuration management
- Real-time network monitoring and alerting
- Configuration backup and compliance checking
- Multi-device orchestration and provisioning
- Intent-based networking automation
- Network troubleshooting and diagnostics

## Architecture

```
+------------------+                    +------------------+
|                  |                    |                  |
|  MCP Client      |<---- MCP Protocol ->|   MCP Server    |
|  (Your App)      |    (HTTP/gRPC)     |                  |
|                  |                    |                  |
+------------------+                    +--------+---------+
                                                 |
                                                 | SSH/NETCONF
                                                 |
                                        +--------v---------+
                                        |                  |
                                        | Cisco Devices    |
                                        |                  |
                                        +------------------+
```

## Security Considerations

- Use environment variables or secret managers for credentials
- Enable TLS/SSL for production gRPC connections
- Implement token-based authentication
- Apply rate limiting to prevent abuse
- Maintain audit logs for all operations
- Use role-based access control (RBAC)

## Contributing

Contributions are welcome! Please feel free to submit issues or pull requests.

## License

This project is provided as-is for educational and reference purposes.

## Resources

- [Official MCP Specification](https://modelcontextprotocol.io)
- [Cisco IOS Command Reference](https://www.cisco.com/c/en/us/support/ios-nx-os-software/ios-15-4m-t/products-command-reference-list.html)
- [Python Netmiko Documentation](https://github.com/ktbyers/netmiko)
- [gRPC Documentation](https://grpc.io/docs/)

## Support

For questions or issues, please open an issue in the GitHub repository.