FROM node:21

WORKDIR /app

COPY . .

RUN npm install --ignore-scripts

EXPOSE 5173

CMD ["npm", "run", "dev", "--", "--host"]
