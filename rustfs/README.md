# RustFs a File System in Rust (S3 compatible)
RustFs is a high-performance, S3-compatible file system implemented in Rust. It provides a reliable and efficient way to store and retrieve files in a distributed environment.
## usage
To use RustFs on Docker, follow these steps:
```bash
# create host directories for data (and logs, optional)
mkdir -p /path/to/rustfs/data /path/to/rustfs/logs

# fix ownership so RustFS inside Docker can write to them
sudo chown -R 1000:1000 /path/to/rustfs/data /data /path/to/rustfs/logs

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
version: "3.8"

services:
  rustfs:
    image: rustfs/rustfs:latest
    container_name: rustfs-server
		restart: unless-stopped
    ports:
      - "9000:9000"   # S3-compatible API
      - "9001:9001"   # Web console
    volumes:
      - ./data:/data   # host ./data → container /data (object storage)
      - ./logs:/logs   # optional: host ./logs → container /logs (for logs)
    environment:
      # (optional) default credentials — change if you like
      - RUSTFS_ACCESS_KEY=rustfsadmin
      - RUSTFS_SECRET_KEY=rustfsadmin
      - RUSTFS_CONSOLE_ENABLE=true
    command: server /data
    # If you want a health-check (optional)
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/rustfs/health/live"]
      interval: 30s
      timeout: 10s
      retries: 3
```