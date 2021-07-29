# dgca-issuance-service-setup-guide
從純淨ubuntu開始安裝
從0開始


## 安裝docker
[Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

## 安裝docker-compose
[Install Docker Compose](https://docs.docker.com/compose/install/)

## 安裝 apache-maven
包裝eu-dgc-issuance-service java專案會用到
### 下載
```bash=
wget https://downloads.apache.org/maven/maven-3/3.8.1/binaries/apache-maven-3.8.1-bin.tar.gz
```
### 解壓縮
```bash=
tar xzvf apache-maven-3.8.1-bin.tar.gz
sudo mv apache-maven-3.8.1 /usr/local
```
### 安裝java jdk
- 先檢查是否有java_home
```bash=
echo $JAVA_HOME
```
>空的就要安裝
- 安裝
```bash=
sudo apt-get update
sudo apt-get install default-jre
sudo apt-get install default-jdk
```
- 設定JAVA_HOME
>[How to set JAVA_HOME in Linux for all users](https://stackoverflow.com/questions/24641536/how-to-set-java-home-in-linux-for-all-users)

### 安裝maven
```bash
export PATH=/usr/local/apache-maven-3.8.1/bin:$PATH
```

## 安裝eu-dgc-issuace-service
[dgca-issuance-service](https://github.com/eu-digital-green-certificates/dgca-issuance-service)

- 建立github personal access token
- `git clone https://github.com/eu-digital-green-certificates/dgca-issuance-service.git`
- 更改`settings.xml`的帳號密碼為github帳號及PAT
- `cp settings.xml ~/.m2/settings.xml` (已經有就不用)
- `sudo docker login docker.pkg.github.com/eu-digital-green-certificates` 輸入你的github帳號及PAT
- 把憑證放到資料夾`certs`裡
- 更改`src/main/resources/application.yml`
```yaml=
issuance:
  dgciPrefix: URN:UVCI:V1:TW
  keyStoreFile: certs/test.jks #改成自己憑證的名字
  keyStorePassword: dgca #改成憑證的密碼
  certAlias: dev_ec #改成憑證的alias
  privateKeyPassword: dgca #改成憑證的密碼
  countryCode: DE #改成TW
  tanExpirationHours: 24 #原本3小時改24小時
  contextFile: context/context.json
  expiration:
    vaccination: 365
    recovery: 365
    test: 3
```
- `mvn clean package -DskipTests`
- 更改`Dockerfiel`
```dockerfile=
FROM adoptopenjdk:11-jre-hotspot
COPY ./target/*.jar /app/app.jar
COPY ./certs /app/certs
COPY ./context/context.json /app/context/context.json
WORKDIR /app
ENTRYPOINT [ "sh", "-c", "java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -jar ./app.jar" ]
```
### docker-compose
- 設定docker-compose
**建議自己重寫一個docker-compose放在所有service的上一層目錄**
```yaml=
version: '3'

services:
  postgres:
    image: library/postgres:9.6
    container_name: dgc-issuance-service-postgres
    #ports:
    #  - 5432:5432
    environment: #記得改帳號密碼
      POSTGRES_DB: postgres
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres 
      restart: unless-stopped
    networks:
      persistence:

  backend:
    build: ./dgca-issuance-service
    image: eu-digital-green-certificates/dgc-issuance-service
    container_name: dgc-issuance-service
    volumes: #記得把certs映射到container
      - ./dgca-issuance-service/certs:/app/certs
    ports:
      - 8080:8080
    environment:
      - SERVER_PORT=8080
      - SPRING_PROFILES_ACTIVE=cloud
      - SPRING_DATASOURCE_URL=jdbc:postgresql://dgc-issuance-service-postgres:5432/postgres
      - SPRING_DATASOURCE_USERNAME=postgres
      - SPRING_DATASOURCE_PASSWORD=postgres
    depends_on:
      - postgres
    networks:
      backend:
      persistence:
    restart: unless-stopped

networks:
  backend:
  persistence:
```
- `sudo docker-compose up`
