# Running OpenVPN Auth OAuth2 Plugin Locally

This guide explains how to run the OpenVPN Auth OAuth2 plugin locally while the OpenVPN servers run in Docker containers.

## Architecture

- **OpenVPN Servers**: Run in Docker containers with exposed management ports
- **Auth Plugin**: Runs locally on your machine
- **Keycloak**: Runs in Docker container for OIDC authentication
- **SQLite Storage**: Local database file for token persistence

## Prerequisites

1. **Go 1.25+** installed locally
2. **Docker and Docker Compose** for running OpenVPN servers and Keycloak
3. **OpenVPN client** for testing connections

## Quick Start

### 1. Start Docker Services

Start the OpenVPN servers and Keycloak:

```bash
# Start OpenVPN servers and Keycloak
docker-compose -f docker-compose.multi-server.yaml up -d

# Wait for services to be ready
docker-compose -f docker-compose.multi-server.yaml logs -f
```

### 2. Build the Plugin Locally

```bash
# Build the plugin
go build -o openvpn-auth-oauth2 .

# Or install it globally
go install .
```

### 3. Run the Plugin Locally

```bash
# Run with local configuration
./openvpn-auth-oauth2 --config config.local.yaml

# Or if installed globally
openvpn-auth-oauth2 --config config.local.yaml
```

## Configuration Details

### Local Configuration (`config.local.yaml`)

Key differences from the Docker configuration:

- **OpenVPN Management**: Uses `localhost:8081` and `localhost:8082` (exposed from Docker)
- **Keycloak**: Uses `localhost:8080` (exposed from Docker)
- **SQLite Path**: Uses local path `./tokens.db`
- **HTTP Listen**: Binds to `localhost:9000`

### Exposed Ports

The Docker Compose setup exposes these ports:

- **OpenVPN Server 1**: 
  - VPN: `1196/tcp`
  - Management: `8081/tcp`
- **OpenVPN Server 2**: 
  - VPN: `1197/tcp`
  - Management: `8082/tcp`
- **Keycloak**: `8080/tcp`

## Testing the Setup

### 1. Verify Services

```bash
# Check Docker services are running
docker-compose -f docker-compose.multi-server.yaml ps

# Test Keycloak connectivity
curl http://localhost:8080/realms/openvpn-auth-oauth2/.well-known/openid-configuration

# Test OpenVPN management ports
telnet localhost 8081
telnet localhost 8082
```

### 2. Test OAuth2 Flow

1. Start the plugin locally: `./openvpn-auth-oauth2 --config config.local.yaml`
2. Visit: http://localhost:9000/oauth2/start
3. Login with: `demo` / `demo123`
4. Verify successful authentication

### 3. Test OpenVPN Connections

```bash
# Copy client configs from Docker containers
docker cp openvpn-server1:/etc/openvpn/client/client-server1.ovpn ./client-server1.ovpn
docker cp openvpn-server2:/etc/openvpn/client/client-server2.ovpn ./client-server2.ovpn

# Connect to server 1
openvpn --config client-server1.ovpn

# Connect to server 2 (in another terminal)
openvpn --config client-server2.ovpn
```

## Development Workflow

### Running in Development Mode

```bash
# Run with debug logging
./openvpn-auth-oauth2 --config config.local.yaml --log-level debug

# Run with hot reload (if using air or similar)
air --config .air.toml
```

### Monitoring

```bash
# View plugin logs
tail -f /var/log/openvpn-auth-oauth2.log

# View Docker logs
docker-compose -f docker-compose.multi-server.yaml logs -f

# Monitor SQLite database
sqlite3 tokens.db "SELECT * FROM tokens;"
```

## Troubleshooting

### Common Issues

1. **Plugin can't connect to OpenVPN management**:
   ```bash
   # Check if management ports are exposed
   netstat -tlnp | grep -E ":(8081|8082)"
   
   # Test management connectivity
   telnet localhost 8081
   ```

2. **Plugin can't connect to Keycloak**:
   ```bash
   # Check Keycloak is running
   docker-compose -f docker-compose.multi-server.yaml ps keycloak
   
   # Test Keycloak connectivity
   curl -v http://localhost:8080/realms/openvpn-auth-oauth2/.well-known/openid-configuration
   ```

3. **Port conflicts**:
   ```bash
   # Check what's using port 9000
   lsof -i :9000
   
   # Kill conflicting processes
   sudo kill -9 $(lsof -t -i:9000)
   ```

### Debug Commands

```bash
# Check plugin process
ps aux | grep openvpn-auth-oauth2

# Check network connections
netstat -tlnp | grep openvpn-auth-oauth2

# View plugin configuration
./openvpn-auth-oauth2 --config config.local.yaml --help

# Test configuration
./openvpn-auth-oauth2 --config config.local.yaml --validate-config
```

## Benefits of Local Plugin Execution

- **Faster Development**: No need to rebuild Docker images
- **Better Debugging**: Direct access to logs and debugging tools
- **IDE Integration**: Full IDE support for debugging and development
- **Hot Reload**: Can use tools like `air` for automatic restarts
- **Local File Access**: Direct access to local files and databases

## Cleanup

```bash
# Stop Docker services
docker-compose -f docker-compose.multi-server.yaml down

# Remove volumes (optional)
docker-compose -f docker-compose.multi-server.yaml down -v

# Clean up local files
rm -f tokens.db
rm -f client-server*.ovpn
```

## Production Considerations

For production deployment:

1. **Use proper secrets management** instead of hardcoded values
2. **Enable TLS** for HTTP endpoints
3. **Use proper logging** with structured logs
4. **Set up monitoring** and health checks
5. **Use process managers** like systemd or supervisor
6. **Implement proper backup** for SQLite database
