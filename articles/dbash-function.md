A simple bash function to quickly access shell environments in Docker containers with automatic shell detection.

### Overview

The `dbash` function provides a convenient way to enter Docker containers with intelligent shell detection. It automatically tries to use `bash` if available, falling back to `sh` if `bash` is not present in the container.

### Function

```bash
dbash() {
    if [ $# -eq 1 ]; then
        if docker exec "$1" which bash >/dev/null 2>&1; then
            docker exec -it "$1" bash
        else
            docker exec -it "$1" sh
        fi
    else
        echo "Usage: dbash <container_name_or_id>"
    fi
}
```

### Features

- **Automatic Shell Detection**: Checks if `bash` is available, falls back to `sh`
- **Simple Usage**: Just provide the container name or ID
- **Interactive Mode**: Uses `-it` flags for proper terminal interaction
- **Error Handling**: Provides usage instructions for incorrect arguments

### Installation

1. Add the function to your shell configuration file:
   - For bash: `~/.bashrc` or `~/.bash_profile`
   - For zsh: `~/.zshrc`

2. Reload your shell configuration:
   ```bash
   source ~/.bashrc  # or ~/.zshrc
   ```

### Usage

```bash
# Enter a container by name
dbash my-container

# Enter a container by ID
dbash a1b2c3d4e5f6

# Invalid usage (shows help)
dbash
dbash container1 container2
```

### Examples

```bash
# Start a container
docker run -d --name nginx-test nginx

# Enter the container
dbash nginx-test

# You're now inside the container with bash or sh
```

### Requirements

- Docker installed and running
- Container must be in running state
- User must have permissions to execute `docker exec`

### How It Works

1. Validates that exactly one argument (container name/ID) is provided
2. Checks if `bash` is available in the target container using `which bash`
3. If `bash` exists, connects using `docker exec -it <container> bash`
4. If `bash` doesn't exist, falls back to `docker exec -it <container> sh`
5. Provides usage instructions if arguments are incorrect

### Benefits

- **Time Saving**: No need to remember different shell commands for different containers
- **Compatibility**: Works with both minimal containers (Alpine, scratch-based) and full distributions
- **Consistency**: Always gets you an interactive shell regardless of container base image

### Related Commands

- `docker exec -it <container> bash` - Direct bash access
- `docker exec -it <container> sh` - Direct sh access  
- `docker ps` - List running containers
- `docker logs <container>` - View container logs
