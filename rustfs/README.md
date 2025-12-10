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


