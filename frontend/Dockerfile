FROM node:21

WORKDIR /app

COPY . .

RUN npm install

RUN npm run test

COPY .env.sample .env.local

EXPOSE 5173

CMD ["npm", "run", "dev", "--", "--host"]
