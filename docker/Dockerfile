FROM node:24-alpine

WORKDIR /usr/src/app

# Copy package files and install only production dependencies
COPY package*.json ./
RUN npm install --omit=dev

# Copy the rest of the app source
COPY . ./

EXPOSE 3000

CMD ["node", "bin/www"]
