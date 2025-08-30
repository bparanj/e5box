# SPIFFE mTLS Demo Guide

This guide demonstrates mutual TLS (mTLS) authentication between a client and server using SPIFFE identities managed by SPIRE.

## Prerequisites

- Docker installed and running
- SPIRE development container built with Go 1.25
- Demo code in `/spire/code/demo/` directory

## Architecture Overview

- **Server**: Listens on port 8443, expects client with SPIFFE ID `spiffe://example.org/client`
- **Client**: Connects to server on port 8443, expects server with SPIFFE ID `spiffe://example.org/server`
- **Authentication**: Based on Unix user IDs (server-workload: 1002, client-workload: 1003)

## Step-by-Step Instructions

### Step 1: Start the Development Container

```bash
# From the spire directory on host
docker run --rm -v "$(pwd)":/spire -it -h spire-dev spire-dev
```

### Step 2: Start SPIRE Infrastructure

Inside the container, check if SPIRE is already running:

```bash
# Check running processes
ps aux | grep spire
```

If not running, start them:

```bash
# Start SPIRE Server
./bin/spire-server run &

# Generate a join token
TOKEN=$(./bin/spire-server token generate -spiffeID spiffe://example.org/host | awk '{print $2}')

# Start SPIRE Agent with the token
./bin/spire-agent run -joinToken $TOKEN &
```

### Step 3: Create Demo Users

Create system users for workload attestation:

```bash
# Create server workload user (UID 1002)
useradd -u 1002 server-workload

# Create client workload user (UID 1003)  
useradd -u 1003 client-workload

# Verify users were created
id server-workload
id client-workload
```

### Step 4: Register SPIFFE IDs

Register the workloads with SPIRE:

```bash
# Register server workload
./bin/spire-server entry create \
  -parentID spiffe://example.org/host \
  -spiffeID spiffe://example.org/server \
  -selector unix:uid:1002

# Register client workload
./bin/spire-server entry create \
  -parentID spiffe://example.org/host \
  -spiffeID spiffe://example.org/client \
  -selector unix:uid:1003

# Verify entries were created
./bin/spire-server entry show
```

Expected output should show both entries with their respective SPIFFE IDs and selectors.

### Step 5: Build the Demo Applications

```bash
# Build the server
cd /spire/code/demo/server
go build -o server server.go

# Build the client
cd /spire/code/demo/client
go build -o client client.go

# Verify binaries were created
ls -la /spire/code/demo/*/server /spire/code/demo/*/client
```

### Step 6: Run the Server

In the current terminal:

```bash
# Navigate to server directory
cd /spire/code/demo/server

# Run server as server-workload user
sudo -u server-workload ./server
```

Expected output:
```
2025/08/30 XX:XX:XX Server starting on :8443
```

### Step 7: Run the Client

Open a new terminal and connect to the container:

```bash
# From host, in spire directory
docker exec -it spire-dev bash

# Inside container, navigate to client directory
cd /spire/code/demo/client

# Run client as client-workload user
sudo -u client-workload ./client
```

Expected output:
```
2025/08/30 XX:XX:XX Connecting to https://localhost:8443/
2025/08/30 XX:XX:XX Success!!!
```

The server terminal should show:
```
2025/08/30 XX:XX:XX Request received
```

## Testing Security

### Test 1: Wrong User Identity

Try running the client as the wrong user:

```bash
# This should FAIL - wrong SPIFFE ID
sudo -u server-workload ./client
```

Expected error:
```
error connecting to "https://localhost:8443/": Get "https://localhost:8443/": remote error: tls: bad certificate
```

### Test 2: No SPIFFE ID

Try running without sudo (as root):

```bash
# This should FAIL - no SPIFFE ID for root
./client
```

Expected error:
```
unable to create X509Source: could not find any x509-SVID
```

## Verify Certificates

Check the SPIFFE certificates being used:

```bash
# View server's certificate
sudo -u server-workload /spire/bin/spire-agent api fetch x509 | head -20

# View client's certificate  
sudo -u client-workload /spire/bin/spire-agent api fetch x509 | head -20
```

## Troubleshooting

### Issue: "connection refused"
- Ensure server is running on port 8443
- Check: `netstat -tlnp | grep 8443`

### Issue: "could not find any x509-SVID"  
- SPIRE agent not running: `ps aux | grep spire-agent`
- Entry not registered: `./bin/spire-server entry show`
- Wait 5-10 seconds for certificate propagation

### Issue: "bad certificate"
- Wrong user running the workload
- Check current user: `whoami`
- Verify SPIFFE ID mapping: `./bin/spire-server entry show`

### Issue: Socket path error
- Verify socket exists: `ls -la /tmp/spire-agent/public/api.sock`
- Check agent logs for errors

## Understanding the Code

### Key Components

1. **Workload API**: Both client and server connect to SPIRE agent via Unix socket at `/tmp/spire-agent/public/api.sock`

2. **X509Source**: Provides automatic certificate rotation and handles SPIFFE identity

3. **TLS Config**:
   - Server uses `tlsconfig.MTLSServerConfig` with `AuthorizeID(clientID)`
   - Client uses `tlsconfig.MTLSClientConfig` with `AuthorizeID(serverID)`

4. **Identity Verification**: Each side validates the other's SPIFFE ID during TLS handshake

## Clean Up

To stop the demo:

```bash
# Kill the server process (Ctrl+C in server terminal)
# Or find and kill the process
pkill -f "demo/server"

# Optionally, stop SPIRE services
pkill spire-server
pkill spire-agent
```

## Next Steps

- Modify the code to use different SPIFFE IDs
- Add additional authorization logic
- Implement certificate rotation handling
- Try other attestation methods (K8s, AWS, etc.)

## Additional Resources

- [SPIFFE Documentation](https://spiffe.io/docs/)
- [go-spiffe Library](https://github.com/spiffe/go-spiffe)
- [SPIRE Documentation](https://spiffe.io/docs/latest/spire/)