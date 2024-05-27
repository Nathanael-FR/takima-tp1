docker run --name takimatp1 -p 5433:5432 -v C:\Users\natha\Desktop\takima-devops\takima-tp1/data:/var/lib/postgresql/data --network app-network  -d nathanaelfr/tp1:v2.0

docker run -p 8090:8080 --net=app-network --name=adminer -d adminer

docker run --name takimatp1 -p 5433:5432 --network app-network  -d nathanaelfr/tp1:v1.0



API

C:\Users\natha\Desktop\takima-devops\takima-tp1\api>docker build -t nathanaelfr/tp1_api:v1.0 .

docker run --name takimatp1_api nathanaelfr/tp1_api:v1.


```
FROM openjdk:11-jre-slim

COPY . /app

WORKDIR /app

CMD [ "java", "Main" ]
```
