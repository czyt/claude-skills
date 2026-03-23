# Docker Run to LazyCat Conversion Examples

This file contains common Docker run command patterns and their LazyCat equivalents.

## 1. Simple Web Server

### Docker Run Command
```bash
docker run -d \
  --name webserver \
  -p 8080:80 \
  -e NGINX_HOST=example.com \
  -v ./html:/usr/share/nginx/html \
  nginx:alpine
```

### LazyCat Files

**lzc-manifest.yml:**
```yaml
lzc-sdk-version: '0.1'
name: Web Server
package: cloud.lazycat.app.webserver
version: 1.0.0
description: "Nginx web server from docker run"

application:
  subdomain: webserver
  routes:
    - /=http://webserver.cloud.lazycat.app.webserver.lzcapp:80

services:
  webserver:
    image: nginx:alpine
    environment:
      - NGINX_HOST=example.com
    binds:
      - /lzcapp/var/html:/usr/share/nginx/html
```

**lzc-build.yml:**
```yaml
manifest: ./lzc-manifest.yml
pkgout: ./
icon: ./icon.png
```

---

## 2. Node.js Application

### Docker Run Command
```bash
docker run -d \
  --name myapp \
  -p 3000:3000 \
  -e NODE_ENV=production \
  -e DATABASE_URL=postgresql://user:pass@db:5432/myapp \
  -v app-data:/app/data \
  --restart unless-stopped \
  myapp/node:latest
```

### LazyCat Files

**lzc-manifest.yml:**
```yaml
lzc-sdk-version: '0.1'
name: My Node App
package: cloud.lazycat.app.myapp
version: 1.0.0
description: "Node.js application from docker run"

application:
  subdomain: myapp
  routes:
    - /=http://myapp.cloud.lazycat.app.myapp.lzcapp:3000

services:
  app:
    image: myapp/node:latest
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://user:pass@db:5432/myapp
    binds:
      - /lzcapp/var/app-data:/app/data
    restart: unless-stopped
    cpu: 1000
    mem_limit: 512M
```

**lzc-build.yml:**
```yaml
manifest: ./lzc-manifest.yml
pkgout: ./
icon: ./icon.png
```

---

## 3. PostgreSQL Database

### Docker Run Command
```bash
docker run -d \
  --name postgres-db \
  -p 5432:5432 \
  -e POSTGRES_DB=myapp \
  -e POSTGRES_USER=user \
  -e POSTGRES_PASSWORD=secretpass \
  -v pgdata:/var/lib/postgresql/data \
  postgres:15
```

### LazyCat Files

**lzc-manifest.yml:**
```yaml
lzc-sdk-version: '0.1'
name: PostgreSQL
package: cloud.lazycat.app.postgres
version: 1.0.0
description: "PostgreSQL database from docker run"

application:
  subdomain: postgres
  ingress:
    - protocol: tcp
      port: 5432
      service: postgres
      description: "PostgreSQL database port"

services:
  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=secretpass
    binds:
      - /lzcapp/var/pgdata:/var/lib/postgresql/data
    cpu: 1000
    mem_limit: 1024M
```

**lzc-build.yml:**
```yaml
manifest: ./lzc-manifest.yml
pkgout: ./
icon: ./icon.png
```

---

## 4. Redis Cache

### Docker Run Command
```bash
docker run -d \
  --name redis-cache \
  -p 6379:6379 \
  -v redis_data:/data \
  redis:7-alpine \
  redis-server --appendonly yes
```

### LazyCat Files

**lzc-manifest.yml:**
```yaml
lzc-sdk-version: '0.1'
name: Redis Cache
package: cloud.lazycat.app.redis
version: 1.0.0
description: "Redis cache from docker run"

application:
  subdomain: redis
  ingress:
    - protocol: tcp
      port: 6379
      service: redis
      description: "Redis cache port"

services:
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    binds:
      - /lzcapp/cache/redis:/data
    cpu: 500
    mem_limit: 256M
```

**lzc-build.yml:**
```yaml
manifest: ./lzc-manifest.yml
pkgout: ./
icon: ./icon.png
```

---

## 5. WordPress Stack

### Docker Run Commands

**Database:**
```bash
docker run -d \
  --name wp-db \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -e MYSQL_DATABASE=wordpress \
  -e MYSQL_USER=wpuser \
  -e MYSQL_PASSWORD=wppass \
  -v wpdb_data:/var/lib/mysql \
  mysql:5.7
```

**WordPress:**
```bash
docker run -d \
  --name wordpress \
  -p 80:80 \
  -e WORDPRESS_DB_HOST=wp-db \
  -e WORDPRESS_DB_USER=wpuser \
  -e WORDPRESS_DB_PASSWORD=wppass \
  -e WORDPRESS_DB_NAME=wordpress \
  -v wp_data:/var/www/html \
  --link wp-db:db \
  wordpress:latest
```

### LazyCat Files

**lzc-manifest.yml:**
```yaml
lzc-sdk-version: '0.1'
name: WordPress Stack
package: cloud.lazycat.app.wordpress-stack
version: 1.0.0
description: "WordPress with MySQL from docker run"

application:
  subdomain: wordpress
  routes:
    - /=http://wordpress.cloud.lazycat.app.wordpress-stack.lzcapp:80

services:
  wordpress:
    image: wordpress:latest
    environment:
      - WORDPRESS_DB_HOST=db
      - WORDPRESS_DB_USER=wpuser
      - WORDPRESS_DB_PASSWORD=wppass
      - WORDPRESS_DB_NAME=wordpress
    binds:
      - /lzcapp/var/wordpress:/var/www/html
    depends_on:
      - db
    cpu: 1000
    mem_limit: 512M

  db:
    image: mysql:5.7
    environment:
      - MYSQL_ROOT_PASSWORD=rootpass
      - MYSQL_DATABASE=wordpress
      - MYSQL_USER=wpuser
      - MYSQL_PASSWORD=wppass
    binds:
      - /lzcapp/var/wpdb:/var/lib/mysql
    cpu: 1000
    mem_limit: 1024M
```

**lzc-build.yml:**
```yaml
manifest: ./lzc-manifest.yml
pkgout: ./
icon: ./icon.png
```

---

## 6. GitLab CE

### Docker Run Command
```bash
docker run -d \
  --name gitlab \
  --hostname gitlab.example.com \
  -p 80:80 -p 443:443 -p 22:22 \
  -e GITLAB_OMNIBUS_CONFIG="external_url 'http://gitlab.example.com'" \
  -v gitlab_config:/etc/gitlab \
  -v gitlab_logs:/var/log/gitlab \
  -v gitlab_data:/var/opt/gitlab \
  --shm-size 256m \
  gitlab/gitlab-ce:latest
```

### LazyCat Files

**lzc-manifest.yml:**
```yaml
lzc-sdk-version: '0.1'
name: GitLab CE
package: cloud.lazycat.app.gitlab-ce
version: 1.0.0
description: "GitLab CE from docker run"

application:
  subdomain: gitlab
  routes:
    - /=http://gitlab.cloud.lazycat.app.gitlab-ce.lzcapp:80
  ingress:
    - protocol: tcp
      port: 22
      service: gitlab
      description: "SSH for Git operations"

services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    environment:
      - GITLAB_OMNIBUS_CONFIG=external_url 'http://gitlab.${LAZYCAT_BOX_DOMAIN}'
    binds:
      - /lzcapp/var/config:/etc/gitlab
      - /lzcapp/var/logs:/var/log/gitlab
      - /lzcapp/var/data:/var/opt/gitlab
    shm_size: '256m'
    cpu: 2000
    mem_limit: 4096M
```

**lzc-build.yml:**
```yaml
manifest: ./lzc-manifest.yml
pkgout: ./
icon: ./icon.png
```

---

## 7. Jenkins CI/CD

### Docker Run Command
```bash
docker run -d \
  --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts
```

### LazyCat Files

**lzc-manifest.yml:**
```yaml
lzc-sdk-version: '0.1'
name: Jenkins
package: cloud.lazycat.app.jenkins
version: 1.0.0
description: "Jenkins CI/CD from docker run"

application:
  subdomain: jenkins
  routes:
    - /=http://jenkins.cloud.lazycat.app.jenkins.lzcapp:8080

services:
  jenkins:
    image: jenkins/jenkins:lts
    binds:
      - /lzcapp/var/jenkins:/var/jenkins_home
    cpu: 2000
    mem_limit: 2048M
```

**lzc-build.yml:**
```yaml
manifest: ./lzc-manifest.yml
pkgout: ./
icon: ./icon.png
```

---

## 8. MinIO Object Storage

### Docker Run Command
```bash
docker run -d \
  --name minio \
  -p 9000:9000 -p 9001:9001 \
  -e MINIO_ROOT_USER=admin \
  -e MINIO_ROOT_PASSWORD=secretpassword \
  -v minio_data:/data \
  minio/minio server /data --console-address ":9001"
```

### LazyCat Files

**lzc-manifest.yml:**
```yaml
lzc-sdk-version: '0.1'
name: MinIO
package: cloud.lazycat.app.minio
version: 1.0.0
description: "MinIO object storage from docker run"

application:
  subdomain: minio
  routes:
    - /=http://minio.cloud.lazycat.app.minio.lzcapp:9001
    - /api/=http://minio.cloud.lazycat.app.minio.lzcapp:9000

services:
  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    environment:
      - MINIO_ROOT_USER=admin
      - MINIO_ROOT_PASSWORD=secretpassword
    binds:
      - /lzcapp/var/minio:/data
    cpu: 1000
    mem_limit: 512M
```

**lzc-build.yml:**
```yaml
manifest: ./lzc-manifest.yml
pkgout: ./
icon: ./icon.png
```

---

## 9. Nginx Proxy Manager

### Docker Run Command
```bash
docker run -d \
  --name nginx-proxy-manager \
  -p 80:80 -p 81:81 -p 443:443 \
  -v npm_data:/data \
  -v npm_letsencrypt:/etc/letsencrypt \
  jc21/nginx-proxy-manager:latest
```

### LazyCat Files

**lzc-manifest.yml:**
```yaml
lzc-sdk-version: '0.1'
name: Nginx Proxy Manager
package: cloud.lazycat.app.npm
version: 1.0.0
description: "Nginx Proxy Manager from docker run"

application:
  subdomain: npm
  routes:
    - /=http://npm.cloud.lazycat.app.npm.lzcapp:81
    - /api/=http://npm.cloud.lazycat.app.npm.lzcapp:81

services:
  nginx-proxy-manager:
    image: jc21/nginx-proxy-manager:latest
    binds:
      - /lzcapp/var/npm-data:/data
      - /lzcapp/var/npm-letsencrypt:/etc/letsencrypt
    cpu: 1000
    mem_limit: 512M
```

**lzc-build.yml:**
```yaml
manifest: ./lzc-manifest.yml
pkgout: ./
icon: ./icon.png
```

---

## 10. Portainer

### Docker Run Command
```bash
docker run -d \
  --name portainer \
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

### LazyCat Files

**lzc-manifest.yml:**
```yaml
lzc-sdk-version: '0.1'
name: Portainer
package: cloud.lazycat.app.portainer
version: 1.0.0
description: "Portainer from docker run"

application:
  subdomain: portainer
  routes:
    - /=http://portainer.cloud.lazycat.app.portainer.lzcapp:9000

services:
  portainer:
    image: portainer/portainer-ce:latest
    binds:
      - /lzcapp/var/portainer:/data
    cpu: 1000
    mem_limit: 512M
```

**lzc-build.yml:**
```yaml
manifest: ./lzc-manifest.yml
pkgout: ./
icon: ./icon.png
```

---

## Quick Reference: Docker Run to LazyCat

| Docker Run Flag | LazyCat Equivalent |
|-----------------|-------------------|
| `--name name` | Service name in `services` |
| `-p 80:80` | HTTP: `routes`<br>TCP/UDP: `ingress` |
| `-e VAR=value` | `environment: ["VAR=value"]` |
| `-v /host:/container` | `binds: ["/lzcapp/var/...:/container"]` |
| `--restart policy` | `restart: policy` |
| `--shm-size size` | `shm_size: size` |
| `--network host` | `network_mode: "host"` |
| `image:tag` | `image: image:tag` |

**Key Rules:**
1. **HTTP ports** → `application.routes`
2. **TCP/UDP ports** → `application.ingress`
3. **Volumes** → `/lzcapp/var` or `/lzcapp/cache`
4. **Environment** → Direct mapping
5. **Resources** → Add `cpu` and `mem_limit`