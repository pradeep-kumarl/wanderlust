FROM node:21
#node js required so we take base image and version

WORKDIR /app
#creates a working directory

copy . .
#copy source to destination

RUN npm i
#install Node Package Manager

EXPOSE 5000
#expose node js port

CMD ["npm", "start"]
#command to run nodejs, write separate like array
