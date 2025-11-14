# Practical Examples for Cisco Device Automation with MCP

This document provides practical examples for using MCP servers and clients to automate Cisco devices.

## Table of Contents

1. [Basic Device Configuration](#basic-device-configuration)
2. [Interface Management](#interface-management)
3. [Multi-Device Operations](#multi-device-operations)
4. [Configuration Backup and Restore](#configuration-backup-and-restore)
5. [Network Monitoring](#network-monitoring)
6. [VLAN Configuration](#vlan-configuration)
7. [Routing Configuration](#routing-configuration)
8. [Security Configuration](#security-configuration)

## Basic Device Configuration

### Example 1: Retrieve Running Configuration

```python
import asyncio
from cisco_mcp_client import CiscoMCPClient

async def get_running_config():
    """Get running configuration from a device"""
    client = CiscoMCPClient("cisco_mcp_server.py")
    
    try:
        await client.connect()
        
        # Retrieve running configuration
        result = await client.get_device_config(
            device_ip="192.168.1.1",
            username="admin",
            password="cisco123",
            config_type="running"
        )
        
        if result["status"] == "success":
            print("Running Configuration:")
            print(result["configuration"])
        else:
            print(f"Error: {result.get('message')}")
    
    finally:
        await client.disconnect()

if __name__ == "__main__":
    asyncio.run(get_running_config())
```

### Example 2: Check Device Version

```python
async def check_device_version():
    """Check IOS version on a device"""
    client = CiscoMCPClient("cisco_mcp_server.py")
    
    try:
        await client.connect()
        
        result = await client.execute_command(
            device_ip="192.168.1.1",
            username="admin",
            password="cisco123",
            command="show version"
        )
        
        if result["status"] == "success":
            output = result["output"]
            # Parse version information
            for line in output.split('\n'):
                if "Version" in line:
                    print(f"IOS Version: {line.strip()}")
    
    finally:
        await client.disconnect()

if __name__ == "__main__":
    asyncio.run(check_device_version())
```

## Interface Management

### Example 3: Configure Interface IP Address

```python
async def configure_interface_ip():
    """Configure IP address on an interface"""
    client = CiscoMCPClient("cisco_mcp_server.py")
    
    try:
        await client.connect()
        
        # Configure GigabitEthernet0/1
        config_commands = [
            "description LAN Interface",
            "ip address 10.1.1.1 255.255.255.0",
            "no shutdown"
        ]
        
        result = await client.configure_interface(
            device_ip="192.168.1.1",
            username="admin",
            password="cisco123",
            interface="GigabitEthernet0/1",
            config_commands=config_commands
        )
        
        if result["status"] == "success":
            print("Interface configured successfully")
            print(result["output"])
        else:
            print(f"Error: {result.get('message')}")
    
    finally:
        await client.disconnect()

if __name__ == "__main__":
    asyncio.run(configure_interface_ip())
```

### Example 4: Monitor Interface Status

```python
async def monitor_interfaces():
    """Monitor all interface status"""
    client = CiscoMCPClient("cisco_mcp_server.py")
    
    try:
        await client.connect()
        
        result = await client.get_interfaces(
            device_ip="192.168.1.1",
            username="admin",
            password="cisco123"
        )
        
        if result["status"] == "success":
            print("Interface Status:")
            print(result["interfaces"])
            
            # Parse and display in a formatted way
            lines = result["interfaces"].split('\n')
            for line in lines:
                if "GigabitEthernet" in line or "FastEthernet" in line:
                    print(line)
        else:
            print(f"Error: {result.get('message')}")
    
    finally:
        await client.disconnect()

if __name__ == "__main__":
    asyncio.run(monitor_interfaces())
```

### Example 5: Shutdown/Enable Interface

```python
async def toggle_interface(interface_name, enable=True):
    """Enable or disable an interface"""
    client = CiscoMCPClient("cisco_mcp_server.py")
    
    try:
        await client.connect()
        
        if enable:
            config_commands = ["no shutdown"]
            action = "enabled"
        else:
            config_commands = ["shutdown"]
            action = "disabled"
        
        result = await client.configure_interface(
            device_ip="192.168.1.1",
            username="admin",
            password="cisco123",
            interface=interface_name,
            config_commands=config_commands
        )
        
        if result["status"] == "success":
            print(f"Interface {interface_name} {action} successfully")
        else:
            print(f"Error: {result.get('message')}")
    
    finally:
        await client.disconnect()

# Usage
if __name__ == "__main__":
    asyncio.run(toggle_interface("GigabitEthernet0/1", enable=True))
```

## Multi-Device Operations

### Example 6: Configure Multiple Devices

```python
async def configure_multiple_devices():
    """Configure multiple devices in parallel"""
    devices = [
        {"ip": "192.168.1.1", "username": "admin", "password": "cisco123"},
        {"ip": "192.168.1.2", "username": "admin", "password": "cisco123"},
        {"ip": "192.168.1.3", "username": "admin", "password": "cisco123"}
    ]
    
    async def configure_single_device(device_info):
        """Configure a single device"""
        client = CiscoMCPClient("cisco_mcp_server.py")
        try:
            await client.connect()
            
            result = await client.execute_command(
                device_ip=device_info["ip"],
                username=device_info["username"],
                password=device_info["password"],
                command="show ip interface brief"
            )
            
            print(f"Device {device_info['ip']}: {result['status']}")
            
        finally:
            await client.disconnect()
    
    # Execute in parallel
    tasks = [configure_single_device(device) for device in devices]
    await asyncio.gather(*tasks)

if __name__ == "__main__":
    asyncio.run(configure_multiple_devices())
```

### Example 7: Device Inventory Collection

```python
async def collect_device_inventory():
    """Collect inventory from multiple devices"""
    devices = [
        {"ip": "192.168.1.1", "name": "Router1"},
        {"ip": "192.168.1.2", "name": "Router2"},
        {"ip": "192.168.1.3", "name": "Switch1"}
    ]
    
    inventory = []
    
    for device in devices:
        client = CiscoMCPClient("cisco_mcp_server.py")
        try:
            await client.connect()
            
            # Get device info
            version_result = await client.execute_command(
                device_ip=device["ip"],
                username="admin",
                password="cisco123",
                command="show version | include Version"
            )
            
            hostname_result = await client.execute_command(
                device_ip=device["ip"],
                username="admin",
                password="cisco123",
                command="show running-config | include hostname"
            )
            
            inventory.append({
                "name": device["name"],
                "ip": device["ip"],
                "version": version_result.get("output", "Unknown"),
                "hostname": hostname_result.get("output", "Unknown")
            })
            
        except Exception as e:
            print(f"Error collecting from {device['name']}: {e}")
        finally:
            await client.disconnect()
    
    # Display inventory
    print("\nDevice Inventory:")
    print("-" * 80)
    for item in inventory:
        print(f"Name: {item['name']}")
        print(f"IP: {item['ip']}")
        print(f"Hostname: {item['hostname']}")
        print(f"Version: {item['version']}")
        print("-" * 80)

if __name__ == "__main__":
    asyncio.run(collect_device_inventory())
```

## Configuration Backup and Restore

### Example 8: Backup Device Configuration

```python
import datetime
import os

async def backup_device_config(device_ip, backup_dir="./backups"):
    """Backup device configuration to a file"""
    client = CiscoMCPClient("cisco_mcp_server.py")
    
    try:
        await client.connect()
        
        # Get running config
        result = await client.get_device_config(
            device_ip=device_ip,
            username="admin",
            password="cisco123",
            config_type="running"
        )
        
        if result["status"] == "success":
            # Create backup directory if it doesn't exist
            os.makedirs(backup_dir, exist_ok=True)
            
            # Create filename with timestamp
            timestamp = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
            filename = f"{backup_dir}/{device_ip}_{timestamp}.cfg"
            
            # Save configuration to file
            with open(filename, 'w') as f:
                f.write(result["configuration"])
            
            print(f"Configuration backed up to: {filename}")
            return filename
        else:
            print(f"Error: {result.get('message')}")
            return None
    
    finally:
        await client.disconnect()

if __name__ == "__main__":
    asyncio.run(backup_device_config("192.168.1.1"))
```

### Example 9: Scheduled Configuration Backup

```python
import asyncio
import schedule
import time

async def scheduled_backup():
    """Backup configurations on a schedule"""
    devices = ["192.168.1.1", "192.168.1.2", "192.168.1.3"]
    
    for device_ip in devices:
        try:
            await backup_device_config(device_ip)
        except Exception as e:
            print(f"Error backing up {device_ip}: {e}")

def run_scheduled_backup():
    """Run the scheduled backup task"""
    asyncio.run(scheduled_backup())

# Schedule backup every day at 2:00 AM
schedule.every().day.at("02:00").do(run_scheduled_backup)

# For testing: backup every 10 minutes
# schedule.every(10).minutes.do(run_scheduled_backup)

print("Backup scheduler started. Press Ctrl+C to stop.")
while True:
    schedule.run_pending()
    time.sleep(60)
```

## Network Monitoring

### Example 10: Real-time Interface Monitoring

```python
async def monitor_interface_continuously(device_ip, interface_name, interval=30):
    """Monitor interface status continuously"""
    client = CiscoMCPClient("cisco_mcp_server.py")
    
    try:
        await client.connect()
        
        print(f"Monitoring {interface_name} on {device_ip}")
        print("Press Ctrl+C to stop\n")
        
        while True:
            result = await client.execute_command(
                device_ip=device_ip,
                username="admin",
                password="cisco123",
                command=f"show interface {interface_name}"
            )
            
            if result["status"] == "success":
                output = result["output"]
                # Extract key metrics
                for line in output.split('\n'):
                    if "input rate" in line or "output rate" in line:
                        print(f"{datetime.datetime.now()}: {line.strip()}")
            
            await asyncio.sleep(interval)
    
    except KeyboardInterrupt:
        print("\nMonitoring stopped")
    finally:
        await client.disconnect()

if __name__ == "__main__":
    asyncio.run(monitor_interface_continuously(
        "192.168.1.1", 
        "GigabitEthernet0/1", 
        interval=30
    ))
```

### Example 11: Check Device Reachability

```python
async def check_device_reachability(devices):
    """Check if devices are reachable"""
    results = []
    
    for device_ip in devices:
        client = CiscoMCPClient("cisco_mcp_server.py")
        try:
            await client.connect()
            
            result = await client.execute_command(
                device_ip=device_ip,
                username="admin",
                password="cisco123",
                command="show clock"
            )
            
            if result["status"] == "success":
                results.append({
                    "ip": device_ip,
                    "status": "reachable",
                    "timestamp": result["output"]
                })
            else:
                results.append({
                    "ip": device_ip,
                    "status": "unreachable",
                    "error": result.get("message")
                })
        
        except Exception as e:
            results.append({
                "ip": device_ip,
                "status": "error",
                "error": str(e)
            })
        finally:
            await client.disconnect()
    
    # Display results
    print("\nDevice Reachability Report:")
    print("-" * 60)
    for result in results:
        print(f"Device: {result['ip']} - Status: {result['status']}")
    print("-" * 60)
    
    return results

if __name__ == "__main__":
    devices = ["192.168.1.1", "192.168.1.2", "192.168.1.3"]
    asyncio.run(check_device_reachability(devices))
```

## VLAN Configuration

### Example 12: Create VLAN

```python
async def create_vlan(device_ip, vlan_id, vlan_name):
    """Create a VLAN on a switch"""
    client = CiscoMCPClient("cisco_mcp_server.py")
    
    try:
        await client.connect()
        
        # Create VLAN using configuration commands
        commands = [
            f"vlan {vlan_id}",
            f"name {vlan_name}",
            "exit"
        ]
        
        result = await client.execute_command(
            device_ip=device_ip,
            username="admin",
            password="cisco123",
            command=f"configure terminal\n" + "\n".join(commands)
        )
        
        if result["status"] == "success":
            print(f"VLAN {vlan_id} ({vlan_name}) created successfully")
            
            # Verify VLAN creation
            verify_result = await client.execute_command(
                device_ip=device_ip,
                username="admin",
                password="cisco123",
                command=f"show vlan id {vlan_id}"
            )
            print("\nVerification:")
            print(verify_result["output"])
        else:
            print(f"Error: {result.get('message')}")
    
    finally:
        await client.disconnect()

if __name__ == "__main__":
    asyncio.run(create_vlan("192.168.1.2", 100, "Engineering"))
```

### Example 13: Assign Interface to VLAN

```python
async def assign_interface_to_vlan(device_ip, interface_name, vlan_id):
    """Assign an interface to a VLAN"""
    client = CiscoMCPClient("cisco_mcp_server.py")
    
    try:
        await client.connect()
        
        config_commands = [
            "switchport mode access",
            f"switchport access vlan {vlan_id}",
            "spanning-tree portfast",
            "no shutdown"
        ]
        
        result = await client.configure_interface(
            device_ip=device_ip,
            username="admin",
            password="cisco123",
            interface=interface_name,
            config_commands=config_commands
        )
        
        if result["status"] == "success":
            print(f"Interface {interface_name} assigned to VLAN {vlan_id}")
        else:
            print(f"Error: {result.get('message')}")
    
    finally:
        await client.disconnect()

if __name__ == "__main__":
    asyncio.run(assign_interface_to_vlan("192.168.1.2", "FastEthernet0/1", 100))
```

## Routing Configuration

### Example 14: Configure Static Route

```python
async def configure_static_route(device_ip, destination, mask, next_hop):
    """Configure a static route"""
    client = CiscoMCPClient("cisco_mcp_server.py")
    
    try:
        await client.connect()
        
        command = f"ip route {destination} {mask} {next_hop}"
        
        result = await client.execute_command(
            device_ip=device_ip,
            username="admin",
            password="cisco123",
            command=f"configure terminal\n{command}"
        )
        
        if result["status"] == "success":
            print(f"Static route to {destination}/{mask} via {next_hop} configured")
            
            # Verify routing table
            verify_result = await client.execute_command(
                device_ip=device_ip,
                username="admin",
                password="cisco123",
                command="show ip route"
            )
            print("\nRouting Table:")
            print(verify_result["output"])
        else:
            print(f"Error: {result.get('message')}")
    
    finally:
        await client.disconnect()

if __name__ == "__main__":
    asyncio.run(configure_static_route(
        "192.168.1.1",
        "10.2.0.0",
        "255.255.0.0",
        "192.168.1.254"
    ))
```

### Example 15: Display Routing Table

```python
async def display_routing_table(device_ip):
    """Display and parse routing table"""
    client = CiscoMCPClient("cisco_mcp_server.py")
    
    try:
        await client.connect()
        
        result = await client.execute_command(
            device_ip=device_ip,
            username="admin",
            password="cisco123",
            command="show ip route"
        )
        
        if result["status"] == "success":
            print(f"Routing Table for {device_ip}:")
            print("=" * 80)
            print(result["output"])
            print("=" * 80)
            
            # Parse and extract static routes
            static_routes = []
            for line in result["output"].split('\n'):
                if line.strip().startswith('S'):
                    static_routes.append(line.strip())
            
            if static_routes:
                print("\nStatic Routes:")
                for route in static_routes:
                    print(f"  {route}")
        else:
            print(f"Error: {result.get('message')}")
    
    finally:
        await client.disconnect()

if __name__ == "__main__":
    asyncio.run(display_routing_table("192.168.1.1"))
```

## Security Configuration

### Example 16: Configure Access Control List (ACL)

```python
async def configure_acl(device_ip, acl_number, rules):
    """Configure an access control list"""
    client = CiscoMCPClient("cisco_mcp_server.py")
    
    try:
        await client.connect()
        
        # Build ACL configuration
        commands = [f"access-list {acl_number} remark Security ACL"]
        commands.extend([f"access-list {acl_number} {rule}" for rule in rules])
        
        config = "configure terminal\n" + "\n".join(commands)
        
        result = await client.execute_command(
            device_ip=device_ip,
            username="admin",
            password="cisco123",
            command=config
        )
        
        if result["status"] == "success":
            print(f"ACL {acl_number} configured successfully")
            
            # Verify ACL
            verify_result = await client.execute_command(
                device_ip=device_ip,
                username="admin",
                password="cisco123",
                command=f"show access-lists {acl_number}"
            )
            print("\nACL Configuration:")
            print(verify_result["output"])
        else:
            print(f"Error: {result.get('message')}")
    
    finally:
        await client.disconnect()

# Usage
if __name__ == "__main__":
    acl_rules = [
        "permit tcp any any eq 443",
        "permit tcp any any eq 80",
        "deny ip any any log"
    ]
    asyncio.run(configure_acl("192.168.1.1", 101, acl_rules))
```

### Example 17: Apply ACL to Interface

```python
async def apply_acl_to_interface(device_ip, interface_name, acl_number, direction="in"):
    """Apply ACL to an interface"""
    client = CiscoMCPClient("cisco_mcp_server.py")
    
    try:
        await client.connect()
        
        config_commands = [
            f"ip access-group {acl_number} {direction}"
        ]
        
        result = await client.configure_interface(
            device_ip=device_ip,
            username="admin",
            password="cisco123",
            interface=interface_name,
            config_commands=config_commands
        )
        
        if result["status"] == "success":
            print(f"ACL {acl_number} applied to {interface_name} ({direction}bound)")
        else:
            print(f"Error: {result.get('message')}")
    
    finally:
        await client.disconnect()

if __name__ == "__main__":
    asyncio.run(apply_acl_to_interface("192.168.1.1", "GigabitEthernet0/0", 101, "in"))
```

## Complete Workflow Example

### Example 18: Complete Device Setup Workflow

```python
async def complete_device_setup(device_ip, hostname, interfaces_config):
    """Complete device setup workflow"""
    client = CiscoMCPClient("cisco_mcp_server.py")
    
    try:
        await client.connect()
        
        print(f"Starting device setup for {device_ip}")
        
        # Step 1: Set hostname
        print("\n1. Setting hostname...")
        hostname_cmd = f"configure terminal\nhostname {hostname}"
        result = await client.execute_command(
            device_ip=device_ip,
            username="admin",
            password="cisco123",
            command=hostname_cmd
        )
        print(f"   Hostname set: {result['status']}")
        
        # Step 2: Configure interfaces
        print("\n2. Configuring interfaces...")
        for interface, config in interfaces_config.items():
            result = await client.configure_interface(
                device_ip=device_ip,
                username="admin",
                password="cisco123",
                interface=interface,
                config_commands=config
            )
            print(f"   {interface}: {result['status']}")
        
        # Step 3: Verify configuration
        print("\n3. Verifying configuration...")
        verify_result = await client.execute_command(
            device_ip=device_ip,
            username="admin",
            password="cisco123",
            command="show running-config | include hostname"
        )
        print(f"   Hostname verification: {verify_result['output']}")
        
        # Step 4: Save configuration
        print("\n4. Saving configuration...")
        save_result = await client.execute_command(
            device_ip=device_ip,
            username="admin",
            password="cisco123",
            command="write memory"
        )
        print(f"   Configuration saved: {save_result['status']}")
        
        print("\nDevice setup completed successfully!")
    
    except Exception as e:
        print(f"Error during setup: {e}")
    finally:
        await client.disconnect()

# Usage
if __name__ == "__main__":
    interfaces = {
        "GigabitEthernet0/0": [
            "description WAN Interface",
            "ip address 203.0.113.1 255.255.255.252",
            "no shutdown"
        ],
        "GigabitEthernet0/1": [
            "description LAN Interface",
            "ip address 192.168.1.1 255.255.255.0",
            "no shutdown"
        ]
    }
    
    asyncio.run(complete_device_setup("192.168.1.1", "Branch-Router-01", interfaces))
```

## Best Practices Demonstrated

1. **Error Handling**: All examples include proper error handling
2. **Resource Management**: Use try-finally blocks to ensure cleanup
3. **Verification**: Verify configuration changes after applying them
4. **Logging**: Print status messages for troubleshooting
5. **Modularity**: Break complex operations into smaller functions
6. **Scalability**: Use async operations for parallel device management
7. **Security**: Never hardcode credentials in production code
8. **Idempotency**: Design operations to be safely retryable

## Additional Resources

- [MCP Overview](./MCP_OVERVIEW.md) - Understanding MCP concepts
- [Server Setup](./SERVER_SETUP.md) - Setting up MCP servers
- [Client Setup](./CLIENT_SETUP.md) - Setting up MCP clients
