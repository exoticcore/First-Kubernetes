# React Application with Nodejs Postgress and NginX

---

<a href='https://www.youtube.com/watch?v=-pTel5FojAQ&t=134s&ab_channel=Codeching'>Tutorail Link</a>

#### Create React App

```
npm create vite@latest
```

<br><br>

#### Create Server dir

```
mkdir server
```

on server dir run:

```
npm init -y
```

install dependencies

```
npm i express nodemon cors pg body-parser
```

create file name keys.js

```
touch key.js
```

`key.js`

```js
module.exports = {
  pgUser: ProcessingInstruction.env.PGUSER,
  pgHost: ProcessingInstruction.env.PGHOST,
  pgDatabase: ProcessingInstruction.env.PGDATABASE,
  pgPassword: ProcessingInstruction.env.PGPASSWORD,
  pgPort: ProcessingInstruction.env.PGPORT,
};
```

Then create index.js file

```
touch index.js
```

`index.js`

```js
const keys = require('./keys');

// Express Application setup
const express = require('express');
const bodyParser = require('body-parser');
const cors = require('cors');

const app = express();
app.use(cors());
app.use(bodyParser.json());

// Potgres client setup
const { Pool } = require('pg');
const pgClient = new Pool({
  user: keys.pgUser,
  host: keys.pgHost,
  database: keys.pgDatabase,
  password: keys.pgPassword,
  port: keys.pgPort,
});

pgClient.on('connect', (client) => {
  client
    .query('CREATE TABLE IF NOT EXISTS values (number INT)')
    .catch((err) => console.log('PG ERROR', err));
});

// Express route definitions
app.get('/', (req, res) => {
  res.send('Hi');
});

app.get('/values/all', async (req, res) => {
  const values = await pgClient.query('SELECT * FROM values');

  res.send(values);
});

// now the post -> insert value
app.post('/values', async (req, res) => {
  if (!req.body.value) res.send({ working: false });

  pgClient.query(`INSERT INTO values(number) VALUES(${req.body.value})`);

  res.send({ working: true });
});

app.listen(5000, (err) => {
  console.log('Listening');
});
```

in `package.json` on scripts

```json
"scripts": {
    "dev": "nodemon",
    "start": "node index.js"
  }
```

<br>

#### At `client` dir

install package

```
npm i axios react-router-dom
```

at `client/src/` create new components

```
touch ./src/OtherPage.jsx
```

`OtherPage.jsx`

```jsx
import React from 'react';
import { Link } from 'react-router-dom';

const OtherPage = () => {
  return (
    <div>
      I'm an other page!<Link to="/">Go back to home screen</Link>
    </div>
  );
};

export default OtherPage;
```

then create `MainComponent.jsx`

```
touch ./src/MainComponent.js
```

`MainComponent.jsx`

```jsx
import React, { useCallback, useState } from 'react';
import axios from 'axios';

const MainComponent = () => {
  const [values, setValues] = useState([]);

  const getAllNumbers = useCallback(async () => {
    // we will use nginx to redirect it to the right request
    const values = await axios.get('/api/values/all');
    setValues(values);
  });

  return (
    <div>
      <button onClick={getAllNumbers}>Get all numbers</button>
      <span className="title">Values</span>
      <div className="values">
        {values.map((value) => (
          <div>{value}</div>
        ))}
      </div>
      <form></form>
    </div>
  );
};

export default MainComponent;
```

`App.js`

```jsx
import './App.css';
import React, { Fragment } from 'react';
import { BrowserRouter as Router, Route, Link } from 'react-router-dom';
import OtherPage from './OtherPage';
import MainComponent from './MainComponent';

function App() {
  return (
    <Router>
      <Fragment>
        <header>
          <div>This is a multicontainer application</div>
          <Link to="/">Home</Link>
          <Link to="/otherpage">Other page</Link>
        </header>
        <div>
          <MainComponent />
          <OtherPage />
          {/* <Route path="/" component={MainComponent} />
          <Route path="/otherpage" component={OtherPage} /> */}
        </div>
      </Fragment>
    </Router>
  );
}

export default App;
```

at `vite-config.js` config:

```js
export default defineConfig({
  plugins: [react()],
  server: {
    watch: {
      usePolling: true,
    },
    host: true, // needed for the Docker Container port mapping to work
    strictPort: true,
    port: 3000,
  },
});
```

creat file in `client`

```
touch Dockerfile.dev
```

`Dockerfile.dev`

```Dockerfile
FROM node:18.6.0-alpine
WORKDIR /app
COPY ./package.json ./
RUN npm i
COPY . .
CMD ["npm", "run", "dev"]
```

then build docker image

```
docker build -t doublecore/multi-client -f Dockerfile.dev .
```

for test it

```
docker run -it -p 4002:3000 doublecore/multi-client
```

then test to run in browser url `http://localhost:4002/`

<br><br>

#### At server

create new Dockerfile.dev

```
touch Dockerfile.dev
```

`Dockerfile.dev` at server dir

```docker
FROM node:18.6.0-alpine
WORKDIR /app
COPY ./package.json ./
RUN npm i
COPY . .
CMD ["npm", "run", "dev"]
```

then build docker image

```
docker build -t doublecore/multi-server -f Dockerfile.dev .
```

try to test its

```
docker run -it -p 4003:5000 doublecore/multi-server
```

<br><br>

#### At main folder

create docker-compose.yaml file

```
touch docker-compose.yaml
```

`docker-compose.yaml`

```yaml
version: '3'
services:
  postgres:
    image: 'postgres:latest'
    environment:
      - POSTGRES_PASSWORD=postgres_password
  api:
    build:
      dockerfile: Dockerfile.dev
      context: './server'
    volumes:
      - /app/node_modules
      - ./server:/app
    environment:
      - PGUSER=postgres
      - PGHOST=postgres
      - PGDATABASE=postgres
      - PGPASSWORD=postgres_password
      - PGPORT=5432
  client:
    stdin_open: true
    build:
      dockerfile: Dockerfile.dev
      context: ./client
    volumes:
      - /app/node_modules
      - ./client:/app
```

then create folder nginx

```
mkdir nginx
```

create file in `./nginx/default.conf` to config NginX server

```
touch ./nginx/default.conf
```

`default.conf`

```.conf
upstream client {
    server client:3000;
}

upstream api {
    server api:5000;
}

server {
    listen 80;

    location / {
        proxy_pass http://client;
    }

    location /sockjs-node {
        proxy_pass http://client;
        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection "Upgrade";
    }

    location /api {
        rewrite /api/(.*) /$1 break;
        proxy_pass http://api;
    }
}
```

after config already create Dockerfile.dev

```
touch ./nginx/Dockerfile.dev
```

in /nginx `Dockerfile.dev`

```
FROM nginx
COPY ./default.conf /etc/nginx/conf.d/default.conf
```

then add nginx to `docker-compose.yaml`

```yaml
nginx:
  depends_on:
    - api
    - client
  restart: always
  build:
    dockerfile: Dockerfile.dev
    context: ./nginx
  port:
    - '3050:80'
```

`docker-compose.yaml` should looks like this

```yaml
version: '3'
services:
  postgres:
    image: 'postgres:latest'
    environment:
      - POSTGRES_PASSWORD=postgres_password
  nginx:
    depends_on:
      - api
      - client
    restart: always
    build:
      dockerfile: Dockerfile.dev
      context: ./nginx
    ports:
      - '3050:80'
  api:
    build:
      dockerfile: Dockerfile.dev
      context: './server'
    volumes:
      - /app/node_modules
      - ./server:/app
    environment:
      - PGUSER=postgres
      - PGHOST=postgres
      - PGDATABASE=postgres
      - PGPASSWORD=postgres_password
      - PGPORT=5432
  client:
    stdin_open: true
    build:
      dockerfile: Dockerfile.dev
      context: ./client
    volumes:
      - /app/node_modules
      - ./client:/app
```

Let's test the docker-compose

```
docker-compose up --build
```

run `http://localhost:3050/` on the browser to test it

<br><br>

#### Let's push containers

Create Dokerfile at `./client`

```
touch Dockerfile
```

then create dir `./client/nginx` to host our vite application

```
mkdir nginx
```

create default.conf at `./client/nginx`

```
touch ./nginx/default.conf
```

`default.conf`

```conf
server{
  listen 3000;

  location / {
    root /usr/share/nginx/html;
    index index.html index.htm;
    try_files $uri $uri/ /index.html;
  }
}
```

`Dockerfile` at `./client`

```Dockerfile
FROM node:18.6.0-alpine AS builder
WORKDIR /app
COPY ./package.json ./
RUN npm i
COPY . .
RUN npm run build

FROM nginx
EXPOSE 3000
COPY ./nginx/default.conf /etc/nginx/conf.d/default.conf
COPY --from=builder /app/dist/ /usr/share/nginx/html
```

then build image

```
docker build -t doublecore/multi-client .
```

try to run an image that we had build

```
docker run -it -p 3000:3000 doublecore/multi-client
```

run `http://localhost:3000` on your browser
If it's show result on the browser, then you're ready to push clien container

```
docker push doublecore/multi-client
```

<br>

##### Let's work on server

create new Dockerfile at `./server`

```
touch Dockerfile
```

`Dockerfile` at `./server`

```Dockerfile
FROM node:18.6.0-alpine
WORKDIR /app
COPY ./package.json ./
RUN npm i
COPY . .
CMD ["npm", "run", "dev"]
```

buid the image at `./server`

```
docker build -t doublecore/multi-server .
```

then run the image

```
docker run -it -p 4002:5000 doublecore/multi-server
```

run `http://localhost:4002` on your brower
push container

```
docker push doublecore/multi-server
```

<br><br><br>

## Kubernetes Multi container Deployment

build folder `k8s`

```
mkdir k8s
```

at `./k8s` create client-deployment

```
touch ./k8s/client-deployment.yaml
```

`client-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: client-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: web
  template:
    metadata:
      labels:
        component: web
    spec:
      containers:
        - name: client
          image: doublecore/multi-client
          ports:
            - containerPort: 3000
```

now we can testing file

```
kubectl apply -f k8s
```

```
kubectl get pods
```

create client service file

```
touch ./k8s/client-cluster-ip-service.yaml
```

`client-cluster-ip-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: client-cluster-ip-service
spec:
  type: ClusterIP
  selector:
    component: web
  ports:
    - port: 3000
      targetPort: 3000
```

then apply

```
kubectl apply -f k8s
```

lists the services

```
kubectl get services
```

we have to create database PVC for postgres

```
touch ./k8s/database-persistent-volume-claim.yaml
```

`database-persistent-volume-cliam.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-persistent-volume-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

create secret generic in kubernetes for postgres password env

```
kubectl create secret generic pgpassword --from-literal PGPASSWORD=12345test
```

get the secrets for checking

```
kubectl get secrets
```

create postgres deployment

```
touch ./k8s/postgres-deployment.yaml
```

`posegres-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: postgres
  template:
    metadata:
      labels:
        component: postgres
    spec:
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: database-persistent-volume-claim
      containers:
        - name: postgres
          image: postgres
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
              subPath: postgres
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: pgpassword
                  key: PGPASSWORD
```

now create postgres service

```
touch ./k8s/postgres-cluster-ip-service.yaml
```

`postgres-cluster-ip-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-cluster-ip-service
spec:
  type: ClusterIP
  selector:
    component: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

then create server deployment

```
touch ./k8s/server-deployment.yaml
```

`server-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: server-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: server
  template:
    metadata:
      labels:
        component: server
    spec:
      containers:
        - name: server
          image: doublecore/multi-server
          ports:
            - containerPort: 5000
          env:
            - name: PGUSER
              value: postgres
            - name: PGHOST
              value: postgres-cluster-ip-service
            - name: PGPORT
              value: '5432'
            - name: PGDATABASE
              value: postgres
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: pgpassword
                  key: PGPASSWORD
```

next, create server service

```
touch ./k8s/server-cluster-ip-service.yaml
```

`server-cluster-ip-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: server-cluster-ip-service
spec:
  type: ClusterIP
  selector:
    component: server
  ports:
    - port: 5000
      targetPort: 3000
```

install controller on this link: `https://kubernetes.github.io/ingress-nginx/deploy/`

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml
```

you can check ingress-nginx on this command

```
kubectl get pods -n ingress-nginx
```

then create ingress service file

```
touch ./k8s/ingress-service.yaml
```

`ingress-service.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-service
  annotations:
    kubernetes.io/ingress.class: 'nginx'
    nginx.ingress.kubernetes.io/user-regex: 'true'
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  rules:
    - http:
        paths:
          - path: /?(.*)
            pathType: Prefix
            backend:
              service:
                name: client-cluster-ip-service
                port:
                  number: 3000
          - path: /api/?(.*)
            pathType: Prefix
            backend:
              service:
                name: server-cluster-ip-service
                port:
                  number: 5000
```

then apply file

```
kubectl apply -f k8s
```

then check on these command:

```
kubectl get pods
```

```
kubectl get pv
```

```
kubectl get services
```

#### minikube for HyperV

start your minikube base on hyperV

```
minikube start --driver=hyperv
```

then enable ingress and ingress-dnc

```
minikube addons enable ingress
```

```
minikube addons enable ingress-dns
```

Wait until you see the ingress-nginx-controller-XXXX is up and running using

```
kubectl get pods -n ingress-nginx
```

Then create Key and update the service section

```
kubectl create secret generic pgpassword --from-literal PGPASSWORD=12345test
```

```
kubectl apply -f k8s
```

then run

```
minikube tunnel
```

request that minikube tunnel giving your IP, it shoud work !!
