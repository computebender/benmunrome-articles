When I was picking a framework for my website, Angular 17 stood out because of its server-side rendering (SSR) features. It's perfect for sites loaded with content, especially when you want them easily found by search engines. Angular with SSR is great, but it does make the deployment a bit more complex than usual. Normally, for a simple Single-Page Application, you don't need much. Something like GitHub Pages does the job, serving up static files. But with SSR, you can't just serve static files since the server has to render pages first.

This left me with two options: using short-life cloud functions or running a long-lived server on a VPS.

I first tried using Vercel for automatic builds and deployment. It was easy to get started, but I knew going in it wasn't fully geared for SSR. Debugging deployment issues turned into a bit of a headache. While I did manage to get around (some) of these problems, the whole experience left me wanting something better. Vercel, not being initially designed for SSR, needed some tweaks and workarounds. Since it's a managed platform, you're kind of stuck with their rules and features. I would consider going back to Vercel if they start supporting SSR more directly.

So, I ended up choosing to run my app on a VPS. It's definitely more work to set up, but you get total control, and honestly, it's a bit of a fun challenge.

## Setting up the CI/CD pipeline

I wanted my deployment process to be super automated. My goals were:

1. Automatically update the site when I change anything in the main branch.
2. Build the Angular app with all the optimizations of a production configuration.
3. Deploy the new version, replacing the old one on the server.
4. Set and forget, no fiddling around after the first setup.
5. Make sure the site is secure with HTTPS.

I decided to go with Docker and Docker Compose on a Digital Ocean Droplet. The major benefit of Docker is that if it works on my machine, it's going to work on the server. The server just needs Docker, no matter what the app needs. And for HTTPS, I picked Caddy as my reverse proxy. Caddy makes setting up HTTPS a breeze, handling certificates with Let's Encrypt automatically. With Docker Compose, I could easily link up everything, keeping the services talking to each other but away from the rest of the web.

### Building the Docker Image

The Dockerfile is the blueprint for my Docker image. It's split into two parts: building and deploying. This way, the final image is lean, packing only what's needed to run the app.

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

#### Build stage

The build stage starts with a node:20-alpine base image, it's a small package but has everything I need. It copies over the package.json and package-lock.json, installs all dependencies, then pulls in the app source and builds it.

#### Run stage

The run stage is also on node:20-alpine, but it only takes what's necessary to run the app from the build stage. It skips all the extra stuff, cutting down on size. Then it gets the app up and running, starting the node service that runs on port 4000.

### Caddy

Caddy's my choice for a web server. It's super user-friendly for solo devs like me. A single config file is all it takes to set up a secure reverse proxy. It really can't get simpler.

```
benmunro.me

reverse_proxy app:4000
```

The latest [Caddyfile](https://github.com/computebender/benmunrome/blob/main/Caddyfile) is available on GitHub.

### Docker Compose

Docker Compose is the glue that holds it all together. It defines the services, sets up a network for them, and makes sure Caddy's data is persisted. It targets the latest app version from GitHub Container Registry as well as the newest Caddy from DockerHub. The setup also makes sure Caddy is ready to handle incoming web traffic.

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

## Automating It!

I'm all about making things easy, so I used GitHub Actions for automation. These are YAML files that lay out the steps to build, publish, and deploy the app. They're triggered by any merges to the main branch. The actions check out the latest code, build and publish a Docker image, and then update the services on the Digital Ocean Droplet with the newest versions.

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
            docker compose pull
            docker compose up -d
```

## Key Takeaways

Sure, this method takes more hands-on effort than just hooking into Vercel, but the control it gives is worth it. I can tweak and adjust to my heart's content. The end result is a self-updating, secure site that takes care of itself after any changes I make. Mission complete!
