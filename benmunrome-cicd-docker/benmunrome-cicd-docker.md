Angular 17 using SSR (server-side rendering) was the framework of choice for this project. It provides some crucial features needed for a content-heavy website that I'd like to ensure is searchable by search engines. For all its excelent features, it does require a more complicated deployment setup. For a traditional Single-Page Application, a fully hosted solution wouldn't generally be needed. There's static hosting options like GitHub Pages that simply return the static application files to the user. Since SSR is being used, this isn't possible as the server needs to first render pages before delivering them to the user. This left me with a few options: using short-life cloud functions, or running a long-lived server on a VPS.

I initially setup the project to use Vercel to automatically build and deploy the project. This worked well initially, but wasnt entirtely designed to be used in this way. I ran into difficult to solve issues debugging the deployment. While these were surmountable, I wasn't pleased with the overall experience. The service wasn't intially built for SSR aplications, and required non-documented configurations to work. As it was a managed platform, you had to play by their rules, within the features of their platform. Perhaps when they more officially support this type of deployment I'll give it another try.

This left me with running the application on a VPS. While significantly more complicated to setup, it provided full control of all steps of the process, plus who doesn't like a challenge?

# Setting up the CI/CD pipeline

Even though I'd never be able to reach the ease of use of Vercel, I knew I wanted the process to be as automated as possible. I had five main goals for the pipeline:

1. Update the deployed site any time a change is made on the main branch.
2. Build the Angular application using the build configuration for the optimizations it provided.
3. Deploy the new version to the server, replacing the existing deployment.
4. Not have to touch anything after initial setup to make this happen.
5. Must be hosted with HTTPS.

I settled on using Docker and Docker Compose running on a Digital Ocean Droplet. Using Docker meant if it ran on my machine, it was going to run on the server. No other dependencies except Docker was required on the server, regardless of application dependencies, this will never change. Requiring HTTPS meant I needed a way to run an additonal reverse proxy service to handle incoming requests. To simplify this I settle on running a Caddy server. Caddy is an great reverse proxy that automates the process of fetching and renuing certificates using Let's Encrypt. To bring it all together Docker Compose defines the two services, and configures a network that allows them to communicate, isolated from the web.

## Building the Docker Image

Docker images are described by a Dockerfile. This file takes a base image, a series of steps to install dependencies, and run the application. The process is split into two stages, a build stage and a deploy stage. This allows the final image to contain only what is necessary for running, and not all build dependencies, reducing its size. The Dockerfile is rather small for all the convenience it provides:

```docker
FROM node:20-alpine as build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
COPY ./ ./
RUN npm run build benmunrome --configuration=production

FROM node:20-alpine

WORKDIR /app
COPY --from=build /app/package.json ./
COPY --from=build /app/angular.json ./

COPY --from=build /app/dist/ ./dist/

CMD ["npm", "run", "serve:ssr:benmunrome"]
```

The latest [Dockerfile](https://github.com/computebender/benmunrome/blob/main/Dockerfile) is available on GitHub.

### Build stage

The first half of the Dockerfile describes the build stage. This stage is based on the node:20-alpine base image. This image provides the v20 Long-Term Support version of node and the associated npm version in small package. The `package.json` and `package-lock.json` files that include build scripts and a list of dependencies are copied to the build container. All dependencies are installed, the app source is copied into the build container, and the application is built.

### Run stage

The second half of the Dockerfile describes the run stage, which is also based on the same image. This stage copies only the required files to run the application. All other dependencies and source code are not needed, therefore are not copied. This stage starts the application, which begins awaiting requests on port 4000.

## Caddy

Caddy is a web server with automatic HTTPS. Compared to other reverse proxy solutions, this is by far the easiest for a solo developer to use. All that is required to configure an HTTPS secured reverse proxy to the application is a single configuration file,

```
benmunro.me

reverse_proxy app:4000
```

The latest [Caddyfile](https://github.com/computebender/benmunrome/blob/main/Caddyfile) is available on GitHub.

It really can't get simpler.

## Docker Compose

Bringing it all together is Docker Compose. It defines how to run the two images as services. It also defines a network to connect the services, and Docker volumes to persist Caddy data.

```yaml
version: "3.8"

services:
  app:
    image: ghcr.io/computebender/benmunrome:latest
    networks:
      - webnet

  caddy:
    image: caddy:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
    networks:
      - webnet
    depends_on:
      - app

networks:
  webnet:

volumes:
  caddy_data:
  caddy_config:
```

The latest [docker-compose.yml](https://github.com/computebender/benmunrome/blob/main/docker-compose.yml) is available on GitHub.

The latest version of the application is referencved from the GitHub Container Registry and the latest version of Caddy is used. The configuration for the Caddy service also exposes ports for incoming requests. Caddy proxies incoming requests on port 80/443 over the Docker network to the server running in the app service.

# Automating It!

While this configuration certainly makes it easier to deploy, I'm lazy and want it to happen automatically. For this I used GitHub Actions. GitHub Actions are yaml files that contain the workflow required to build, publish, and deploy an application. Triggers can also be defined that start the workflow automatically on different actions within GitHub, in this case on any merge to the `main` branch.

```yaml
name: Build and Deploy

on:
  push:
    branches:
      - main

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Check out the code
        uses: actions/checkout@v2

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v1
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository_owner }}/benmunrome:latest

  deploy:
    runs-on: ubuntu-latest
    needs: build-and-push

    steps:
      - name: Deploy to VPS
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.DO_HOST }}
          username: ${{ secrets.DO_USER }}
          key: ${{ secrets.DO_PRIVATE_KEY }}
          script: |
            cd /var/www/benmunrome
            git pull origin main
            docker compose down
            docker compose up --pull -d
```

The workflow files first checks out the latest code from the repository. It then logins in to my GitHub container registry using secrets provided by the running environment. A Docker image is build and published according to the Dockerfile. The final stage connects to the Digital Ocean Droplet, pulls the latest configuration, are restarts the services using the latest version of each.

# Conclusion

While this setup is certainly more effort than simply connecting to Vercel, it gives me full control over the entire process. I can customize it in pretty much any way I'd like. In the end I end up with a similar result, upon any change my website is automatically re-deployed to a secure server without any interaction on my part. Mission complete!
