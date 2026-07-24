# Fix Gitea Runner Docker Hub Rate Limits

<img src="/articles/images/docker-logo.svg" alt="Docker" width="200">

### Problem

Gitea runners hitting Docker Hub rate limits when pulling images:


> Error response from daemon: toomanyrequests: You have reached your unauthenticated pull rate limit


### Solution: Mount Docker Config File

Mount the Docker config file directly into the runner container:

```yaml
volumes:
  - ./docker-config.json:/root/.docker/config.json:ro
```

### How It Works

**File Structure:**

```
Your Project Directory:
├── docker-compose.yml
├── docker-config.json  ← Authentication file
└── .env
```

**Mount Explanation:**

- `./docker-config.json` - File on your host machine
- `/root/.docker/config.json` - Where Docker expects auth config in container
- `:ro` - Read-only mount (container can't modify the file)

### Step-by-Step Setup

#### 1. Create Docker Hub Personal Access Token
- Go to [https://hub.docker.com](https://hub.docker.com)
- Account Settings → Security → Personal Access Tokens
- Create token with "Public Repo Read-only" permissions

<img src="/articles/images/docker-access-token.png" alt="docker-access-token" width="800"/>

#### 2. Generate Base64 Auth String

```bash
echo -n "your-docker-username:your-docker-token" | base64
```

#### 3. Create docker-config.json

```json
{
  "auths": {
    "https://index.docker.io/v1/": {
      "auth": "your-base64-string-here"
    }
  }
}
```

#### 4. Update docker-compose.yml

```yaml
version: '3.8'
services:
  gitea-runner:
    image: gitea/act_runner:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - gitea-runner-data:/data
      - ./docker-config.json:/root/.docker/config.json:ro  # ← This line
```

#### 5. Restart Runner

```bash
docker-compose down
docker-compose up -d
```

### Verification

Check logs for successful authentication:

```bash
docker-compose logs -f gitea-runner
```
