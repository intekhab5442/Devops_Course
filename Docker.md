# Dockerfile Notes — Languages & Frameworks

Below is a practical **Dockerfile reference for commonly used languages and frameworks**. The goal is not to memorize every Dockerfile, but to understand the **pattern behind each ecosystem**.

---

# 1. Dockerfile Fundamentals

A Dockerfile is a text file containing instructions used to build a Docker image.

### Basic structure

```dockerfile
FROM <base-image>

WORKDIR /app

COPY <files> .

RUN <command>

EXPOSE <port>

CMD ["command"]
```

### Most important instructions

| Instruction   | Purpose                                           |
| ------------- | ------------------------------------------------- |
| `FROM`        | Selects the base image                            |
| `WORKDIR`     | Sets working directory                            |
| `COPY`        | Copies files into image                           |
| `ADD`         | Copies files; also has extra archive/URL behavior |
| `RUN`         | Executes command while building                   |
| `CMD`         | Default command when container starts             |
| `ENTRYPOINT`  | Defines the main executable                       |
| `EXPOSE`      | Documents container port                          |
| `ENV`         | Sets environment variable                         |
| `ARG`         | Build-time variable                               |
| `USER`        | Changes container user                            |
| `VOLUME`      | Declares mount point                              |
| `HEALTHCHECK` | Defines container health check                    |

### `RUN` vs `CMD` vs `ENTRYPOINT`

```dockerfile
RUN npm install
```

Runs **during image build**.

```dockerfile
CMD ["npm", "start"]
```

Runs **when the container starts**.

```dockerfile
ENTRYPOINT ["python"]
```

Defines the main executable of the container.

---

# 2. `.dockerignore`

Always consider using `.dockerignore`.

Example:

```text
node_modules
.git
.env
*.log
coverage
dist
.vscode
.idea
```

It prevents unnecessary files from entering the build context.

---

# 3. Node.js

Common frameworks:

* Express
* NestJS
* Next.js
* Fastify
* Koa

## Basic Node.js Dockerfile

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```

### Why `COPY package*.json` first?

Docker caches layers.

If your source code changes but `package.json` doesn't, Docker can reuse the dependency-installation layer.

### `npm install` vs `npm ci`

For Docker/CI environments, generally prefer:

```dockerfile
RUN npm ci
```

`npm ci` is designed for clean, reproducible installs using the lock file.

For production:

```dockerfile
RUN npm ci --omit=dev
```

---

# 4. Node.js Multi-Stage Dockerfile

```dockerfile
FROM node:22-alpine AS builder

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

RUN npm run build


FROM node:22-alpine

WORKDIR /app

COPY package*.json ./

RUN npm ci --omit=dev

COPY --from=builder /app/dist ./dist

EXPOSE 3000

CMD ["node", "dist/index.js"]
```

### Architecture

```text
Source Code
     ↓
Builder Image
     ↓
npm install
     ↓
Build
     ↓
dist/
     ↓
Runtime Image
     ↓
Application
```

---

# 5. Express.js

Express normally doesn't require a special base image.

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev

COPY . .

EXPOSE 3000

CMD ["node", "server.js"]
```

---

# 6. NestJS

NestJS generally needs a build stage.

```dockerfile
FROM node:22-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

RUN npm run build


FROM node:22-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev

COPY --from=builder /app/dist ./dist

EXPOSE 3000

CMD ["node", "dist/main.js"]
```

---

# 7. Next.js

For Next.js, a production Docker image can use standalone output.

`next.config.js`:

```javascript
module.exports = {
  output: "standalone"
};
```

Dockerfile:

```dockerfile
FROM node:22-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

RUN npm run build


FROM node:22-alpine AS runner

WORKDIR /app

ENV NODE_ENV=production

COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static

EXPOSE 3000

CMD ["node", "server.js"]
```

---

# 8. Python

Common frameworks:

* Django
* Flask
* FastAPI
* Celery

## Basic Python

```dockerfile
FROM python:3.13-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["python", "app.py"]
```

### Why `--no-cache-dir`?

```dockerfile
RUN pip install --no-cache-dir -r requirements.txt
```

Prevents pip from retaining its package cache and can reduce image size.

---

# 9. Flask

```dockerfile
FROM python:3.13-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["python", "app.py"]
```

For production, Flask is commonly run behind a WSGI server such as Gunicorn:

```dockerfile
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
```

---

# 10. FastAPI

FastAPI applications commonly use Uvicorn.

```dockerfile
FROM python:3.13-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

For multiple workers, the deployment configuration may differ depending on your environment.

---

# 11. Django

```dockerfile
FROM python:3.13-slim

WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["gunicorn", "project.wsgi:application", "--bind", "0.0.0.0:8000"]
```

---

# 12. Java

Java commonly uses:

* Maven
* Gradle
* Spring Boot
* Quarkus
* Micronaut

The major concept to understand is:

```text
JDK → Build application
JRE/JVM runtime → Run application
```

## Maven Java Application

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS builder

WORKDIR /app

COPY pom.xml .

RUN mvn dependency:go-offline

COPY src ./src

RUN mvn clean package -DskipTests


FROM eclipse-temurin:21-jre

WORKDIR /app

COPY --from=builder /app/target/*.jar app.jar

EXPOSE 8080

CMD ["java", "-jar", "app.jar"]
```

This is a classic **multi-stage build**.

---

# 13. Spring Boot

Spring Boot commonly produces a JAR.

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS builder

WORKDIR /app

COPY pom.xml .

RUN mvn dependency:go-offline

COPY src ./src

RUN mvn clean package -DskipTests


FROM eclipse-temurin:21-jre

WORKDIR /app

COPY --from=builder /app/target/*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

# 14. Java Gradle

```dockerfile
FROM gradle:8-jdk21 AS builder

WORKDIR /app

COPY build.gradle* settings.gradle* ./

COPY . .

RUN gradle build --no-daemon -x test


FROM eclipse-temurin:21-jre

WORKDIR /app

COPY --from=builder /app/build/libs/*.jar app.jar

EXPOSE 8080

CMD ["java", "-jar", "app.jar"]
```

---

# 15. PHP

Common frameworks:

* Laravel
* Symfony
* CodeIgniter

PHP commonly uses:

```text
PHP-FPM + Nginx
```

rather than running PHP directly as the web server in production.

## Basic PHP

```dockerfile
FROM php:8.4-apache

WORKDIR /var/www/html

COPY . .

RUN docker-php-ext-install pdo pdo_mysql

EXPOSE 80
```

The official PHP image provides helper scripts such as:

```text
docker-php-ext-install
```

for PHP extensions.

---

# 16. Laravel

Laravel commonly needs Composer.

```dockerfile
FROM composer:2 AS builder

WORKDIR /app

COPY composer.json composer.lock ./

RUN composer install \
    --no-dev \
    --optimize-autoloader \
    --no-interaction

COPY . .


FROM php:8.4-cli

WORKDIR /app

COPY --from=builder /app /app

RUN docker-php-ext-install pdo pdo_mysql

EXPOSE 8000

CMD ["php", "artisan", "serve", "--host=0.0.0.0", "--port=8000"]
```

For production, Laravel is normally deployed with a proper PHP-FPM/web-server setup rather than relying on `artisan serve`.

---

# 17. Go

Go is particularly well suited to multi-stage builds because it can compile into a single binary.

```dockerfile
FROM golang:1.24-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./

RUN go mod download

COPY . .

RUN go build -o server .


FROM alpine:3.22

WORKDIR /app

COPY --from=builder /app/server .

EXPOSE 8080

CMD ["./server"]
```

### Concept

```text
Go source
   ↓
Go compiler
   ↓
Single binary
   ↓
Small runtime image
```

This can produce very small production images.

---

# 18. Rust

Rust is also excellent for multi-stage builds.

```dockerfile
FROM rust:1.89 AS builder

WORKDIR /app

COPY Cargo.toml Cargo.lock ./

RUN mkdir src && \
    echo "fn main() {}" > src/main.rs

RUN cargo build --release

COPY src ./src

RUN cargo build --release


FROM debian:bookworm-slim

WORKDIR /app

COPY --from=builder /app/target/release/app .

EXPOSE 8080

CMD ["./app"]
```

For optimized Rust images, you can also investigate distroless or minimal runtime approaches depending on your application's dependencies.

---

# 19. C

C applications need a compiler during the build.

```dockerfile
FROM gcc:15 AS builder

WORKDIR /app

COPY . .

RUN gcc main.c -o app


FROM debian:bookworm-slim

WORKDIR /app

COPY --from=builder /app/app .

CMD ["./app"]
```

---

# 20. C++

For C++:

```dockerfile
FROM gcc:15 AS builder

WORKDIR /app

COPY . .

RUN g++ main.cpp -o app


FROM debian:bookworm-slim

WORKDIR /app

COPY --from=builder /app/app .

CMD ["./app"]
```

For larger projects, CMake is generally used.

---

# 21. CMake C/C++ Project

```dockerfile
FROM gcc:15 AS builder

RUN apt-get update && \
    apt-get install -y cmake && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY . .

RUN cmake -S . -B build && \
    cmake --build build


FROM debian:bookworm-slim

WORKDIR /app

COPY --from=builder /app/build/myapp .

CMD ["./myapp"]
```

---

# 22. C#

Modern C# applications commonly use:

* .NET
* ASP.NET Core
* Entity Framework

The important distinction is:

```text
SDK image → build
ASP.NET runtime image → run web application
```

## ASP.NET Core

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS builder

WORKDIR /src

COPY *.csproj ./

RUN dotnet restore

COPY . .

RUN dotnet publish -c Release -o /app/publish


FROM mcr.microsoft.com/dotnet/aspnet:10.0

WORKDIR /app

COPY --from=builder /app/publish .

EXPOSE 8080

ENTRYPOINT ["dotnet", "MyApp.dll"]
```

The exact major version should match the .NET version your project targets.

---

# 23. Ruby

Common frameworks:

* Ruby on Rails
* Sinatra

Basic Ruby:

```dockerfile
FROM ruby:3.4-slim

WORKDIR /app

COPY Gemfile Gemfile.lock ./

RUN bundle install

COPY . .

EXPOSE 3000

CMD ["ruby", "app.rb"]
```

---

# 24. Ruby on Rails

```dockerfile
FROM ruby:3.4-slim

WORKDIR /app

RUN apt-get update && \
    apt-get install -y build-essential libpq-dev && \
    rm -rf /var/lib/apt/lists/*

COPY Gemfile Gemfile.lock ./

RUN bundle install

COPY . .

EXPOSE 3000

CMD ["bin/rails", "server", "-b", "0.0.0.0"]
```

Production Rails deployments usually require additional configuration for assets, databases, secrets and the application server.

---

# 25. Kotlin

Kotlin applications often use Gradle.

```dockerfile
FROM gradle:8-jdk21 AS builder

WORKDIR /app

COPY . .

RUN gradle build --no-daemon


FROM eclipse-temurin:21-jre

WORKDIR /app

COPY --from=builder /app/build/libs/*.jar app.jar

EXPOSE 8080

CMD ["java", "-jar", "app.jar"]
```

---

# 26. Scala

Scala commonly uses SBT.

```dockerfile
FROM sbtscala/scala-sbt:latest AS builder

WORKDIR /app

COPY . .

RUN sbt clean assembly


FROM eclipse-temurin:21-jre

WORKDIR /app

COPY --from=builder /app/target/scala-*/myapp.jar app.jar

CMD ["java", "-jar", "app.jar"]
```

The actual artifact path depends on the SBT project configuration.

---

# 27. Dart

Dart applications can also be containerized.

```dockerfile
FROM dart:stable AS builder

WORKDIR /app

COPY pubspec.* ./

RUN dart pub get

COPY . .

RUN dart compile exe bin/server.dart -o bin/server


FROM debian:bookworm-slim

WORKDIR /app

COPY --from=builder /app/bin/server ./server

EXPOSE 8080

CMD ["./server"]
```

---

# 28. Flutter Web

Flutter mobile applications are not normally packaged into Docker containers for running on a server.

But **Flutter Web** can be built and served using Nginx.

```dockerfile
FROM ghcr.io/cirruslabs/flutter:stable AS builder

WORKDIR /app

COPY . .

RUN flutter pub get

RUN flutter build web --release


FROM nginx:alpine

COPY --from=builder /app/build/web /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

---

# 29. React

React is a frontend framework/library that is normally **built with Node.js** and then served by Nginx.

```dockerfile
FROM node:22-alpine AS builder

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

RUN npm run build


FROM nginx:alpine

COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### Important architecture

```text
React source
     ↓
Node.js
     ↓
npm run build
     ↓
Static files
     ↓
Nginx
     ↓
Browser
```

---

# 30. Angular

```dockerfile
FROM node:22-alpine AS builder

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

RUN npm run build


FROM nginx:alpine

COPY --from=builder /app/dist/<project-name>/browser /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

The output directory varies with the Angular project's configuration/version.

---

# 31. Vue.js

```dockerfile
FROM node:22-alpine AS builder

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

RUN npm run build


FROM nginx:alpine

COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

Vue follows essentially the same containerization pattern as React.

---

# 32. Svelte / SvelteKit

For a static Svelte application:

```dockerfile
FROM node:22-alpine AS builder

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

RUN npm run build


FROM nginx:alpine

COPY --from=builder /app/build /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

SvelteKit can have different deployment adapters, so the runtime depends on whether you're deploying static output, Node, or another target.

---

# 33. WordPress

A common starting point:

```dockerfile
FROM wordpress:php8.4-apache

COPY . /var/www/html/
```

However, WordPress deployments usually use the official WordPress image together with a separate database container such as MySQL or MariaDB.

---

# 34. Nginx

If you only need a static website:

```dockerfile
FROM nginx:alpine

COPY ./html /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

---

# 35. Static HTML/CSS/JS

You don't need Node.js if your website is already static.

```dockerfile
FROM nginx:alpine

COPY . /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

---

# 36. Elixir / Phoenix

Elixir applications commonly use Mix and Phoenix.

A production build is typically multi-stage:

```dockerfile
FROM hexpm/elixir:1.18-erlang-27-alpine AS builder

WORKDIR /app

RUN mix local.hex --force && \
    mix local.rebar --force

COPY mix.exs mix.lock ./

RUN mix deps.get --only prod

COPY . .

RUN MIX_ENV=prod mix compile

RUN MIX_ENV=prod mix release


FROM alpine:3.22

WORKDIR /app

COPY --from=builder /app/_build/prod/rel/my_app ./

EXPOSE 4000

CMD ["bin/my_app", "start"]
```

---

# 37. Haskell

Haskell commonly uses GHC/Cabal or Stack.

Conceptually:

```dockerfile
FROM haskell:9 AS builder

WORKDIR /app

COPY . .

RUN cabal update
RUN cabal build


FROM debian:bookworm-slim

WORKDIR /app

COPY --from=builder /app/dist-newstyle/build/.../app .

CMD ["./app"]
```

The exact binary path depends on the Cabal project.

---

# 38. Perl

```dockerfile
FROM perl:5.40

WORKDIR /app

COPY cpanfile .

RUN cpanm --installdeps .

COPY . .

CMD ["perl", "app.pl"]
```

---

# 39. R

For R applications:

```dockerfile
FROM rocker/r-ver:4.4

WORKDIR /app

COPY . .

RUN R -e "install.packages(c('plumber'), repos='https://cloud.r-project.org')"

EXPOSE 8000

CMD ["R", "-e", "pr <- plumber::plumb('plumber.R'); pr$run(host='0.0.0.0', port=8000)"]
```

---

# 40. Common Framework Pattern

You can simplify most Dockerfiles into a few categories.

### Compiled language

Examples:

```text
Go
Rust
C
C++
Java
C#
```

Pattern:

```text
Source
 ↓
Compiler / SDK
 ↓
Binary / JAR
 ↓
Small runtime image
```

Dockerfile:

```dockerfile
FROM builder-image AS builder

# Install dependencies
# Copy source
# Compile

FROM runtime-image

# Copy compiled application

CMD [...]
```

---

### Interpreted/runtime language

Examples:

```text
Python
Ruby
PHP
Node.js
```

Pattern:

```text
Runtime image
 ↓
Install dependencies
 ↓
Copy source
 ↓
Start application
```

---

### Frontend framework

Examples:

```text
React
Angular
Vue
Svelte
Flutter Web
```

Pattern:

```text
Node/Flutter
     ↓
Build
     ↓
Static files
     ↓
Nginx
```

---

# 41. Dockerfile Best Practices

### 1. Use a suitable base image

Instead of:

```dockerfile
FROM ubuntu
```

prefer a language-specific or minimal runtime image when appropriate:

```dockerfile
FROM node:22-alpine
```

---

### 2. Use multi-stage builds

Instead of shipping compilers and development tools:

```text
Builder
   ↓
Compile
   ↓
Runtime
```

---

### 3. Use `.dockerignore`

```text
.git
node_modules
.env
*.log
```

---

### 4. Don't put secrets inside Dockerfiles

Bad:

```dockerfile
ENV DB_PASSWORD=mysecretpassword
```

Prefer runtime secret/configuration mechanisms.

---

### 5. Don't run applications as root when practical

Example:

```dockerfile
USER node
```

or create a dedicated non-root user appropriate for the base image.

---

### 6. Combine related package-manager commands

For Debian/Ubuntu-based images:

```dockerfile
RUN apt-get update && \
    apt-get install -y curl git && \
    rm -rf /var/lib/apt/lists/*
```

This helps avoid leaving package indexes behind in the image layer.

---

### 7. Order instructions for caching

Prefer:

```dockerfile
COPY package*.json ./
RUN npm ci

COPY . .
```

instead of:

```dockerfile
COPY . .
RUN npm ci
```

---

# 42. `CMD` vs `ENTRYPOINT`

### CMD

```dockerfile
CMD ["node", "app.js"]
```

Can be overridden:

```bash
docker run image python app.py
```

### ENTRYPOINT

```dockerfile
ENTRYPOINT ["node"]
```

Then:

```bash
docker run image app.js
```

runs:

```text
node app.js
```

### Together

```dockerfile
ENTRYPOINT ["python"]
CMD ["app.py"]
```

This gives:

```bash
docker run image
```

→ `python app.py`

and:

```bash
docker run image test.py
```

→ `python test.py`

---

# 43. `ARG` vs `ENV`

### ARG

Available primarily during image build:

```dockerfile
ARG APP_VERSION=1.0

RUN echo $APP_VERSION
```

Build:

```bash
docker build --build-arg APP_VERSION=2.0 .
```

### ENV

Available in the resulting image/container:

```dockerfile
ENV NODE_ENV=production
```

Container:

```bash
echo $NODE_ENV
```

---

# 44. `COPY` vs `ADD`

Generally prefer:

```dockerfile
COPY . .
```

Use `ADD` only when you specifically need its additional behavior.

For ordinary file copying:

```text
COPY > ADD
```

---

# 45. Port Configuration

If your application listens on:

```text
8080
```

you can document it:

```dockerfile
EXPOSE 8080
```

But `EXPOSE` **does not publish the port**.

You still need:

```bash
docker run -p 8080:8080 image
```

Meaning:

```text
Host 8080 → Container 8080
```

---

# 46. Healthcheck

Example:

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s \
    CMD curl -f http://localhost:8080/health || exit 1
```

This allows Docker to determine whether the container is healthy.

---

# 47. Most Important Multi-Stage Pattern

Memorize this pattern:

```dockerfile
FROM <builder> AS builder

WORKDIR /app

COPY dependency-files ./

RUN install-dependencies

COPY . .

RUN build-application


FROM <runtime>

WORKDIR /app

COPY --from=builder /app/output .

EXPOSE <port>

CMD ["start", "application"]
```

You can adapt this to:

```text
Node.js
Java
Go
Rust
C
C++
.NET
React
Angular
Vue
Flutter Web
```

---

# 48. Quick Revision Table

| Technology     | Build Image        | Runtime         |  Common Port |
| -------------- | ------------------ | --------------- | -----------: |
| Node.js        | Node               | Node            |         3000 |
| Express        | Node               | Node            |         3000 |
| NestJS         | Node               | Node            |         3000 |
| Next.js        | Node               | Node            |         3000 |
| Python         | Python             | Python          |         8000 |
| Flask          | Python             | Python/Gunicorn |         5000 |
| FastAPI        | Python             | Uvicorn         |         8000 |
| Django         | Python             | Gunicorn        |         8000 |
| Java           | Maven/Gradle + JDK | JRE/JVM         |         8080 |
| Spring Boot    | Maven/Gradle       | JRE/JVM         |         8080 |
| PHP            | PHP                | PHP/Apache/FPM  |      80/9000 |
| Laravel        | Composer + PHP     | PHP-FPM/Apache  |      8000/80 |
| Go             | Go                 | Minimal Linux   |         8080 |
| Rust           | Rust               | Minimal Linux   |         8080 |
| C              | GCC                | Linux runtime   | App-specific |
| C++            | GCC/CMake          | Linux runtime   | App-specific |
| C#             | .NET SDK           | ASP.NET runtime |         8080 |
| Ruby           | Ruby               | Ruby            |         3000 |
| Rails          | Ruby               | Ruby            |         3000 |
| Kotlin         | Gradle/JDK         | JVM             |         8080 |
| Scala          | SBT                | JVM             |         8080 |
| React          | Node               | Nginx           |           80 |
| Angular        | Node               | Nginx           |           80 |
| Vue            | Node               | Nginx           |           80 |
| Svelte         | Node               | Nginx/Node      |      80/3000 |
| Flutter Web    | Flutter            | Nginx           |           80 |
| WordPress      | PHP                | Apache/PHP      |           80 |
| Static HTML    | None               | Nginx           |           80 |
| Elixir/Phoenix | Elixir             | Alpine          |         4000 |

---

# 49. What You Should Actually Memorize

As a DevOps engineer, **don't try to memorize 30 Dockerfiles word-for-word**.

Memorize these five patterns:

```text
1. Runtime application
2. Compiled application
3. Multi-stage build
4. Frontend → Nginx
5. Production optimization
```

Then learn how each ecosystem changes:

```text
Node.js → npm ci
Python  → pip install
Java    → Maven/Gradle
PHP     → Composer
Go      → go build
Rust    → cargo build
C/C++   → gcc/g++/cmake
.NET    → dotnet restore/publish
Ruby    → bundle install
Frontend → npm build → Nginx
```

This is the approach that will make you capable of writing a Dockerfile for a **new technology you haven't previously seen**, rather than merely reproducing memorized examples.
