# Multi-Server OpenVPN Auth OAuth2 Testing Setup

This setup demonstrates the new multiple OpenVPN server support with SQLite token storage.

## Architecture

The setup includes:
- **2 OpenVPN Servers**: `openvpn-server1` (TCP port 1196) and `openvpn-server2` (TCP port 1197)
- **OpenVPN Auth OAuth2 Plugin**: Manages authentication for both servers
- **Keycloak**: OIDC provider for authentication
- **SQLite Storage**: Persistent token storage across restarts

## Quick Start

1. **Start the services**:
   ```bash
   docker-compose -f docker-compose.multi-server.yaml up -d
   ```

2. **Wait for services to be ready** (about 30-60 seconds):
   ```bash
   docker-compose -f docker-compose.multi-server.yaml logs -f
   ```

3. **Access the services**:
   - **Keycloak Admin**: http://localhost:8080 (admin/insecure)
   - **Auth Plugin**: http://localhost:9000
   - **Debug Interface**: http://localhost:9001

## Testing Authentication

### 1. Test OAuth2 Flow

Visit http://localhost:9000/oauth2/start in your browser:
- You'll be redirected to Keycloak
- Login with: `demo` / `demo123`
- You'll be redirected back to the auth plugin

### 2. Test OpenVPN Connection

The OpenVPN servers will generate client configuration files:
- **Server 1**: `client-server1.ovpn` (connects to TCP port 1196)
- **Server 2**: `client-server2.ovpn` (connects to TCP port 1197)

To test with OpenVPN client:
```bash
# Copy client config from container
docker cp openvpn-server1:/etc/openvpn/client/client-server1.ovpn ./client-server1.ovpn
docker cp openvpn-server2:/etc/openvpn/client/client-server2.ovpn ./client-server2.ovpn

# Connect to server 1
openvpn --config client-server1.ovpn

# In another terminal, connect to server 2
openvpn --config client-server2.ovpn
```

### 3. Test Multiple Server Management

The auth plugin manages both servers simultaneously:
- Check logs: `docker-compose -f docker-compose.multi-server.yaml logs openvpn-auth-oauth2`
- Monitor connections: `docker-compose -f docker-compose.multi-server.yaml logs openvpn-server1 openvpn-server2`

## Configuration Details

### Multi-Server Configuration

The `config.multi-server.yaml` file configures:
- **SQLite Token Storage**: Persistent across restarts
- **Two OpenVPN Servers**: Different management ports and passwords
- **Shared OAuth2 Settings**: Same Keycloak provider for both servers

### Server-Specific Settings

Each OpenVPN server has:
- **Different Network Ranges**: 10.8.0.0/23 and 10.8.2.0/23
- **Different Management Ports**: 8081 and 8082
- **Different Passwords**: password1 and password2
- **Different TCP Ports**: 1196 and 1197

## Monitoring and Debugging

### View Logs
```bash
# All services
docker-compose -f docker-compose.multi-server.yaml logs -f

# Specific service
docker-compose -f docker-compose.multi-server.yaml logs -f openvpn-auth-oauth2
```

### Check SQLite Database
```bash
# Access the auth plugin container
docker exec -it openvpn-auth-oauth2 sh

# View token storage
sqlite3 /var/lib/openvpn-auth-oauth2/tokens.db ".tables"
sqlite3 /var/lib/openvpn-auth-oauth2/tokens.db "SELECT * FROM tokens;"
```

### Test Management Interface
```bash
# Connect to server 1 management
docker exec -it openvpn-server1 telnet localhost 8081

# Connect to server 2 management
docker exec -it openvpn-server2 telnet localhost 8082
```

## Cleanup

```bash
# Stop and remove all containers
docker-compose -f docker-compose.multi-server.yaml down

# Remove volumes (this will delete the SQLite database)
docker-compose -f docker-compose.multi-server.yaml down -v
```

## Troubleshooting

### Common Issues

1. **Services not starting**: Check if TCP ports 1196, 1197, 8080, 9000, 9001 are available
2. **Authentication failing**: Verify Keycloak is healthy and accessible
3. **OpenVPN connection issues**: Check firewall rules and network configuration

### Debug Commands

```bash
# Check service health
docker-compose -f docker-compose.multi-server.yaml ps

# Check network connectivity
docker exec -it openvpn-auth-oauth2 ping openvpn-server1
docker exec -it openvpn-auth-oauth2 ping openvpn-server2

# Check Keycloak health
curl http://localhost:8080/realms/openvpn-auth-oauth2/.well-known/openid-configuration
```

## Features Demonstrated

This setup demonstrates:
- ✅ Multiple OpenVPN server management
- ✅ SQLite token storage with cleanup
- ✅ Server-specific authentication flows
- ✅ Shared OAuth2 provider configuration
- ✅ Persistent token storage across restarts
- ✅ Management interface connectivity
- ✅ Network isolation between servers
