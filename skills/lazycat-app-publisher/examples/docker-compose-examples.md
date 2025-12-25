# Docker Compose to LazyCat Conversion Examples

This file contains common Docker Compose patterns and their LazyCat equivalents.

## 1. Simple Web Application

### Docker Compose
```yaml
version: '3'
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html
    environment:
      - NGINX_HOST=example.com
```

### LazyCat Files

**lzc-manifest.yml:**
```yaml
lzc-sdk-version: '0.1'
name: Nginx Web
package: cloud.lazycat.app.nginx-web
version: 1.0.0
description: "Simple Nginx web server"

application:
  subdomain: nginxweb
  routes:
    - /=http://nginxweb.cloud.lazycat.app.nginx-web.lzcapp:80

services:
  web:
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

## 2. WordPress with Database

### Docker Compose
```yaml
version: '3.8'
services:
  wordpress:
    image: wordpress:latest
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wordpress_data:/var/www/html
    depends_on:
      - db

  db:
    image: mysql:5.7
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
      MYSQL_ROOT_PASSWORD: rootpassword
    volumes:
      - db_data:/var/lib/mysql

volumes:
  wordpress_data:
  db_data:
```

### LazyCat Files

**lzc-manifest.yml:**
```yaml
lzc-sdk-version: '0.1'
name: WordPress
package: cloud.lazycat.app.wordpress
version: 1.0.0
description: "WordPress with MySQL database"

application:
  subdomain: wordpress
  routes:
    - /=http://wordpress.cloud.lazycat.app.wordpress.lzcapp:80

services:
  wordpress:
    image: wordpress:latest
    environment:
      - WORDPRESS_DB_HOST=db
      - WORDPRESS_DB_USER=wordpress
      - WORDPRESS_DB_PASSWORD=wordpress
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
      - MYSQL_DATABASE=wordpress
      - MYSQL_USER=wordpress
      - MYSQL_PASSWORD=wordpress
      - MYSQL_ROOT_PASSWORD=rootpassword
    binds:
      - /lzcapp/var/mysql:/var/lib/mysql
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

## 3. GitLab with SSH Access

### Docker Compose
```yaml
version: '3.6'
services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    ports:
      - "80:80"
      - "443:443"
      - "22:22"
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://gitlab.example.com'
    volumes:
      - ./config:/etc/gitlab
      - ./logs:/var/log/gitlab
      - ./data:/var/opt/gitlab
    shm_size: '256m'
```

### LazyCat Files

**lzc-manifest.yml:**
```yaml
lzc-sdk-version: '0.1'
name: GitLab CE
package: cloud.lazycat.app.gitlab-ce
version: 1.0.0
description: "GitLab Community Edition"

application:
  subdomain: gitlab
  routes:
    - /=http://gitlab.cloud.lazycat.app.gitlab-ce.lzcapp:80
    - /api/=http://gitlab.cloud.lazycat.app.gitlab-ce.lzcapp:80
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

## 4. Node.js with Redis

### Docker Compose
```yaml
version: '3.8'
services:
  app:
    image: node:18-alpine
    working_dir: /app
    command: npm start
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - REDIS_URL=redis://cache:6379
    volumes:
      - ./src:/app
    depends_on:
      - cache

  cache:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

volumes:
  redis_data:
```

### LazyCat Files

**lzc-manifest.yml:**
```yaml
lzc-sdk-version: '0.1'
name: Node.js App
package: cloud.lazycat.app.nodejs-app
version: 1.0.0
description: "Node.js application with Redis cache"

application:
  subdomain: nodejsapp
  routes:
    - /=http://nodejsapp.cloud.lazycat.app.nodejs-app.lzcapp:3000

services:
  app:
    image: node:18-alpine
    command: npm start
    working_dir: /app
    environment:
      - NODE_ENV=production
      - REDIS_URL=redis://cache:6379
    binds:
      - /lzcapp/var/src:/app
    depends_on:
      - cache
    cpu: 1000
    mem_limit: 512M

  cache:
    image: redis:7-alpine
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

## 5. Portainer (Management UI)

### Docker Compose
```yaml
version: '3.8'
services:
  portainer:
    image: portainer/portainer-ce:latest
    ports:
      - "9000:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    restart: unless-stopped

volumes:
  portainer_data:
```

### LazyCat Files

**lzc-manifest.yml:**
```yaml
lzc-sdk-version: '0.1'
name: Portainer
package: cloud.lazycat.app.portainer
version: 1.0.0
description: "Portainer Container Management"

application:
  subdomain: portainer
  routes:
    - /=http://portainer.cloud.lazycat.app.portainer.lzcapp:9000

services:
  portainer:
    image: portainer/portainer-ce:latest
    binds:
      - /lzcapp/var/data:/data
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

## 6. MongoDB Database

### Docker Compose
```yaml
version: '3.8'
services:
  mongodb:
    image: mongo:6
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: example
    volumes:
      - mongodb_data:/data/db

volumes:
  mongodb_data:
```

### LazyCat Files

**lzc-manifest.yml:**
```yaml
lzc-sdk-version: '0.1'
name: MongoDB
package: cloud.lazycat.app.mongodb
version: 1.0.0
description: "MongoDB Database"

application:
  subdomain: mongodb
  ingress:
    - protocol: tcp
      port: 27017
      service: mongodb
      description: "MongoDB database port"

services:
  mongodb:
    image: mongo:6
    environment:
      - MONGO_INITDB_ROOT_USERNAME=root
      - MONGO_INITDB_ROOT_PASSWORD=example
    binds:
      - /lzcapp/var/mongodb:/data/db
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

## 7. Nextcloud with Collaborative Features

### Docker Compose
```yaml
version: '3.8'
services:
  nextcloud:
    image: nextcloud:latest
    ports:
      - "8080:80"
    environment:
      MYSQL_PASSWORD: nextcloud
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_HOST: db
    volumes:
      - nextcloud_data:/var/www/html
    depends_on:
      - db

  db:
    image: mariadb:10.6
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_PASSWORD: nextcloud
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
    volumes:
      - db_data:/var/lib/mysql

volumes:
  nextcloud_data:
  db_data:
```

### LazyCat Files

**lzc-manifest.yml:**
```yaml
lzc-sdk-version: '0.1'
name: Nextcloud
package: cloud.lazycat.app.nextcloud
version: 1.0.0
description: "Nextcloud File Hosting"

application:
  subdomain: nextcloud
  routes:
    - /=http://nextcloud.cloud.lazycat.app.nextcloud.lzcapp:8080

services:
  nextcloud:
    image: nextcloud:latest
    environment:
      - MYSQL_PASSWORD=nextcloud
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=db
    binds:
      - /lzcapp/var/nextcloud:/var/www/html
    depends_on:
      - db
    cpu: 1500
    mem_limit: 1024M

  db:
    image: mariadb:10.6
    environment:
      - MYSQL_ROOT_PASSWORD=rootpassword
      - MYSQL_PASSWORD=nextcloud
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
    binds:
      - /lzcapp/var/mariadb:/var/lib/mysql
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

## Quick Reference: Conversion Rules

| Docker Compose | LazyCat Equivalent |
|----------------|-------------------|
| `image: name:tag` | `image: name:tag` (same) |
| `ports: ["80:80"]` | HTTP: `routes: ["/=http://...:80"]`<br>TCP/UDP: `ingress: [{protocol: tcp, port: 80}]` |
| `volumes: ["./data:/app"]` | `binds: ["/lzcapp/var/data:/app"]` |
| `environment: VAR=value` | `environment: ["VAR=value"]` (same) |
| `depends_on: ["db"]` | `depends_on: ["db"]` (same) |
| `restart: always` | `restart: always` (same) |
| `shm_size: 256m` | `shm_size: 256m` (same) |
| `network_mode: host` | `network_mode: "host"` (same) |

**Important**: Always use `/lzcapp/var` or `/lzcapp/cache` for persistent data paths!