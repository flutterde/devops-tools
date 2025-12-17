# RustFs a File System in Rust (S3 compatible)
RustFs is a high-performance, S3-compatible file system implemented in Rust. It provides a reliable and efficient way to store and retrieve files in a distributed environment.
## usage
To use RustFs on Docker, follow these steps:
```bash
# create host directories for data (and logs, optional)
mkdir -p /path/to/rustfs/data /path/to/rustfs/logs

# fix ownership so RustFS inside Docker can write to them
sudo chown -R 1000:1000 /path/to/rustfs/data /data /path/to/rustfs/logs
sudo chmod -R 777 /path/to/rustfs/data /data /path/to/rustfs/logs

cd /path/to/rustfs/

# run RustFs container
docker run -d \
	--name rustfs_cont \
	-p 9000:9000 -p 9001:9001 \
	-v $(pwd)/data:/data \
	-v $(pwd)/logs:/logs \  # optional
	rustfs/rustfs:latest
```

## Configuration
RustFs can be configured using environment variables. Here are some common configuration options:
- `RUSTFS_ACCESS_KEY`: Set the access key for authentication.
- `RUSTFS_SECRET_KEY`: Set the secret key for authentication.
- `RUSTFS_CONSOLE_ENABLE`: Enable or disable the web console (default: true).
- `RUSTFS_CONSOLE_CORS_ALLOWED_ORIGINS`: Set allowed origins for CORS in the web console.
- `RUSTFS_ADDRESS`: Port address for RustFs to listen on (default: 9000).
- `RUSTFS_CONSOLE_ADDRESS`: Port address for the web console (default: 9001).


## Use `docker compose`
You can also use `docker compose` to run RustFs. Here is an example `docker-compose.yml` file:
```yaml

services:
  rustfs:
    image: rustfs/rustfs:latest
    container_name: rustfs
    restart: unless-stopped
    ports:
      - "9000:9000"
      - "9001:9001"
    networks:
      - default
    volumes:
      - rustfs_data:/data
      - rustfs_logs:/logs
    environment:
      - RUSTFS_ACCESS_KEY=${RUSTFS_ACCESS_KEY:adminrustfs}
      - RUSTFS_SECRET_KEY=${RUSTFS_SECRET_KEY:adminrustfs}
      - RUSTFS_CONSOLE_ENABLE=${RUSTFS_CONSOLE_ENABLE:true}

networks:
  default:
    driver: bridge

volumes:
  rustfs_data:
  rustfs_logs:
```

# Migrate from MinIO to RustFs
To migrate data from MinIO to RustFs, you can use the the following script:

let say this is your docker-compose.yml.

```yaml
services:

  minio:
    image: minio/minio
    container_name: minio
    restart: unless-stopped
    ports:
      - "9002:9000"
      - "9003:9001"
    networks:
      - default
    volumes:
      - ./minio/data:/data
    environment:
      - MINIO_ACCESS_KEY=${MINIO_ACCESS_KEY}
      - MINIO_SECRET_KEY=${MINIO_SECRET_KEY}
    command: server /data --console-address ":9001"

  rustfs:
    image: rustfs/rustfs:latest
    container_name: rustfs
    restart: unless-stopped
    ports:
      - "9000:9000"
      - "9001:9001"
    networks:
      - default
    volumes:
      - rustfs_data:/data
      - rustfs_logs:/logs
    environment:
      - RUSTFS_ACCESS_KEY=${RUSTFS_ACCESS_KEY}
      - RUSTFS_SECRET_KEY=${RUSTFS_SECRET_KEY}
      - RUSTFS_CONSOLE_ENABLE=${RUSTFS_CONSOLE_ENABLE}

networks:
  default:
    driver: bridge

volumes:
  rustfs_data:
  rustfs_logs:

```

run it, then execute the following script to migrate data:
```bash
#!/bin/bash

# Migration script: MinIO to RustFS
# This script copies all data from MinIO to RustFS using mc (MinIO Client)

set -e

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
NC='\033[0m'

echo -e "${YELLOW}=== MinIO to RustFS Migration Script ===${NC}"
echo ""

# MinIO configuration (source)
echo -e "${YELLOW}Enter MinIO (source) configuration:${NC}"
read -p "MinIO Endpoint [http://localhost:9002]: " MINIO_ENDPOINT
MINIO_ENDPOINT=${MINIO_ENDPOINT:-http://localhost:9002}

read -p "MinIO Access Key: " MINIO_ACCESS_KEY
read -sp "MinIO Secret Key: " MINIO_SECRET_KEY
echo ""

# RustFS configuration (destination)
echo ""
echo -e "${YELLOW}Enter RustFS (destination) configuration:${NC}"
read -p "RustFS Endpoint [http://localhost:9000]: " RUSTFS_ENDPOINT
RUSTFS_ENDPOINT=${RUSTFS_ENDPOINT:-http://localhost:9000}

read -p "RustFS Access Key: " RUSTFS_ACCESS_KEY
read -sp "RustFS Secret Key: " RUSTFS_SECRET_KEY
echo ""
echo ""

# Check if mc is installed, otherwise use temporary location
MC_CMD="mc"
if ! command -v mc &> /dev/null; then
    echo -e "${YELLOW}MinIO Client (mc) not found. Downloading temporarily...${NC}"
    
    # Download mc to /tmp (will be removed after script ends)
    wget -q https://dl.min.io/client/mc/release/linux-amd64/mc -O /tmp/mc
    chmod +x /tmp/mc
    MC_CMD="/tmp/mc"
    
    echo -e "${GREEN}MinIO Client downloaded to /tmp${NC}"
fi

echo -e "${YELLOW}Configuring MinIO Client aliases...${NC}"

# Configure aliases for both servers
$MC_CMD alias set minio_source "$MINIO_ENDPOINT" "$MINIO_ACCESS_KEY" "$MINIO_SECRET_KEY" 2>/dev/null || true
$MC_CMD alias set rustfs_dest "$RUSTFS_ENDPOINT" "$RUSTFS_ACCESS_KEY" "$RUSTFS_SECRET_KEY" 2>/dev/null || true

echo -e "${GREEN}Aliases configured${NC}"

# List buckets in MinIO (source)
echo -e "${YELLOW}Listing buckets in MinIO...${NC}"
$MC_CMD ls minio_source 2>/dev/null || {
    echo -e "${RED}Error: Cannot connect to MinIO. Make sure MinIO is running.${NC}"
    echo -e "${YELLOW}If MinIO is not running, start it first:${NC}"
    echo "  docker compose -f docker-compose-dev.yml up -d minio"
    echo ""
    echo -e "${YELLOW}Or update MINIO_ENDPOINT in this script to match your MinIO URL${NC}"
    # Cleanup before exit
    [ -f /tmp/mc ] && rm -f /tmp/mc
    exit 1
}

# Get list of buckets
BUCKETS=$($MC_CMD ls minio_source --json 2>/dev/null | grep -o '"key":"[^"]*"' | cut -d'"' -f4 | tr -d '/')

if [ -z "$BUCKETS" ]; then
    echo -e "${YELLOW}No buckets found in MinIO${NC}"
    # Cleanup before exit
    [ -f /tmp/mc ] && rm -f /tmp/mc
    exit 0
fi

echo -e "${GREEN}Found buckets: ${BUCKETS}${NC}"

# Migrate each bucket
for BUCKET in $BUCKETS; do
    echo -e "${YELLOW}Processing bucket: ${BUCKET}${NC}"
    
    # Create bucket in RustFS if it doesn't exist
    echo -e "  Creating bucket in RustFS..."
    $MC_CMD mb "rustfs_dest/${BUCKET}" 2>/dev/null || echo -e "  ${YELLOW}Bucket already exists or created${NC}"
    
    # Mirror (copy) all objects from MinIO to RustFS
    echo -e "  Copying objects..."
    $MC_CMD mirror --overwrite "minio_source/${BUCKET}" "rustfs_dest/${BUCKET}"
    
    echo -e "${GREEN}  Bucket ${BUCKET} migrated successfully${NC}"
done

# Cleanup: Remove mc from /tmp
if [ -f /tmp/mc ]; then
    rm -f /tmp/mc
    echo -e "${YELLOW}Cleaned up: removed /tmp/mc${NC}"
fi

echo ""
echo -e "${GREEN}=== Migration Complete ===${NC}"
echo ""
echo -e "${YELLOW}Verification (run manually if needed):${NC}"
echo "  wget -q https://dl.min.io/client/mc/release/linux-amd64/mc -O /tmp/mc && chmod +x /tmp/mc"
echo "  /tmp/mc alias set rustfs <RUSTFS_ENDPOINT> <ACCESS_KEY> <SECRET_KEY>"
echo "  /tmp/mc ls rustfs"
echo "  rm /tmp/mc"

```