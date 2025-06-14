 SETUP INSTRUCTIONS
Prerequisites
launch AWS ec2 instance (ubuntu server; t2-medium)
Install Docker:

sudo apt update && sudo apt upgrade -y
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
 
## Log out and log back in or run: newgrp docker
Install Docker Compose:

sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
Clone the repository and build:

git clone https://github.com/icnoka/fullstack-todo-list.git
cd fullstack-todo-list
## Dockerfiles
cd Frontend ---> vi Dockerfile --->
## Frontend Dockerfile (Frontend/Dockerfile)
FROM node:18-alpine as builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
ARG VITE_API_URL
ENV VITE_API_URL=$VITE_API_URL
RUN npm run build
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
***********************************************************************************************************************
cd Backend  ---> vi Dockerfile
### Backend Dockerfile (Backend/Dockerfile)
FROM node:18-alpine
WORKDIR /app
COPY backend/package*.json ./
RUN npm install
COPY backend .
EXPOSE 3000
CMD ["node", "index.js"]

***************************************************************************************************************
## Docker Compose Configuration (docker-compose.yml)
version: "3.8"
services:
 mongo:
 image: mongo:6
 container_name: mongodb
 ports:
 - "27017:27017"
 volumes:
 - mongo-data:/data/db
 networks:
 - todo-network
 backend:
 build:
 context: ./Backend
 dockerfile: Dockerfile
 container_name: backend
 ports:
 - "3000:3000"
 depends_on:
 - mongo
 environment:
 - MONGO_URI=mongodb://mongo:27017
 networks:
 - todo-network
 frontend:
 build:
 context: ./Frontend
 dockerfile: Dockerfile
 args:
 VITE_API_URL: http://3.144.175.188:3000
 container_name: frontend
 ports:
 - "80:80"
 depends_on:
 - backend
 networks:
 - todo-network
volumes:
 mongo-data:
networks:
 todo-network:
 driver: bridge
*******************************************************************************************************************************
docker-compose up --build -d
docker ps

Stop the containers:
docker-compose down

********* Network and Security Configurations***********
Network
All services are on a custom Docker bridge network named todo-network.

Containers can communicate via container names (mongo, backend, frontend).

Ports
Service	Container Port	Host Port
Frontend	80	80
Backend	3000	3000
MongoDB	27017	27017

Environment Variables
Frontend uses:


VITE_API_URL=http://<ec2-ip>:3000
This allows the frontend to talk to the backend from any host machine.

Backend uses:

MONGO_URI=mongodb://mongo:27017
This connects to the MongoDB service within the Docker network.

--***No sensitive database credentials are configured. 

******** Troubleshooting Guide
 Issue: Frontend loads but input spins without result
Cause: Frontend  pointing to localhost instead of the EC2 public IP.

Fix: Set VITE_API_URL to http://<EC2-IP>:3000 and rebuild the frontend.

MongoDB connection fails
Fixes:

Ensure Mongo container is up: docker ps

Check logs: docker-compose logs mongo

Ensure backend uses mongodb://mongo:27017 as the URI (not localhost or IP).

 Backend not receiving requests
Check:

That it’s listening on 0.0.0.0: app.listen(PORT, '0.0.0.0')

That the port is exposed: EXPOSE 3000 in Dockerfile
Changes not reflected
Run:

docker-compose down
docker-compose up --build -d

****** Container Testing Script
You can run the following commands to verify the setup:

****** Check if containers are running

docker ps
*****Test backend API endpoint

curl http://<EC2-IP>:3000/api/gettodos
****** Check MongoDB from within backend

docker exec -it backend sh

# Inside container
apk add mongodb-tools
mongosh --host mongo --port 27017
******** Check frontend accessibility
Open browser:
http://<EC2-IP>
You should see the frontend UI.

![Screenshot (532)](https://github.com/user-attachments/assets/b315df20-b7d4-46af-8d91-92089bfd6bf3)

