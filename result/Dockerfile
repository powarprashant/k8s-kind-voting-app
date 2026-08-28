FROM node:22-slim

# add curl for healthcheck; also apply latest OS security patches
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends curl tini && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /usr/local/app

COPY package*.json ./

RUN npm ci && \
 npm cache clean --force && \
 mv /usr/local/app/node_modules /node_modules && \
 # npm/npx are not needed to run the app; drop them (and npm's bundled,
 # CVE-affected 'tar') to keep the runtime image minimal
 rm -rf /usr/local/lib/node_modules/npm \
        /usr/local/bin/npm \
        /usr/local/bin/npx

COPY . .

ENV PORT 80
EXPOSE 80

ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["node", "server.js"]
