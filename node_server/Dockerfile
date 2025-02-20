#docker image to build a nodejs server
#use the latest image of node to build from

FROM node:22

#create working directory for app to run from
WORKDIR /usr/src/app

#Install aoo dependencies
#the * is used to ensure both package.json AND package-lock.json are copied
#where  available (npm@5+)

COPY package*.json ./

RUN npm install

#bundle app source code into docker image i.e. copy all code required to working directory
COPY . .

#as the app binds to port 8080, so expose
EXPOSE 8080

#run command to run app
CMD [ "node", "server.js" ]
