FROM node:21

WORKDIR /app

COPY . .

RUN npm install

COPY .env.sample .env.local

EXPOSE 5173

CMD ["npm", "run", "dev", "--", "--host"]
