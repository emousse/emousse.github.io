+++
title = "Nextjs Strapi Docker"
date = "2023-04-13T11:04:19+02:00"
author = "Etienne Moussé"
authorTwitter = "emousse" #do not include @
cover = "nextjs-strapi-docker.png"
tags = ["nextjs", "strapi", "docker"]
keywords = ["nextjs", "strapi", "docker"]
description = "Monter une app web avec Next.js et strapi avec docker Dans cet article, nous allons résumer nos échanges avec un IA, en mettant en lumière les étapes clés de la création d’un projet Next.js avec Strapi et Docker"
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

# Monter une app web avec Next.js et strapi avec docker

L’objectif de se projet est de ce familiariser avec des technos que je connais sur le papier mais que je n’ai pas encore éprouvé, je suis développeur fullstack JS (react, nodejs) avec une affinité pour le devops.

Je vais donc pas à pas monter ce blog, qui sera à terme un vrai projet en production (je vous en parle bientôt)

Je vous présente rapidement la stack que nous allons mettre en place :

- Un container avec un front Next.js
- Un container avec un back simple nodejs et express
- Un container avec un back strapi
- Un container avec une base de données postgres

Le container communiqueront grâce au network de docker créé par défaut

## **Création de la structure du projet**

D'accord, revenons à la première proposition de stack avec Next.js, PostgreSQL et une API Express. Voici les étapes pour commencer à mettre en place votre projet :

- Créez un nouveau dossier pour votre projet et créez les sous-dossiers pour l'application Next.js, l'API Express et Strapi :

```
Copy code
mon-blog/
├── app/
├── api/
└── strapi/
```

On peut créer tout les répertoires en une seule commande

```bash
mkdir -p mon-blog/{app,api,strapi}
```

1. Initialisation de l'application Next.js :
    - Accédez au dossier **`app`** et exécutez **`npx create-next-app`** pour créer une nouvelle application Next.js.
    - Installez les dépendances nécessaires :
        
        ```bash
        npm install --save next-auth tailwindcss postcss autoprefixer 
        ```
        
2. Configuration de l'API Express :
    - Accédez au dossier **`api`** et exécutez **`npm init -y`** pour initialiser un nouveau projet Node.js.
    - Installez les dépendances nécessaires :
        
        ```bash
        npm install --save express cors body-parser pg sequelize
        ```
        
    - Créez les dossiers et fichiers nécessaires pour définir les routes, les contrôleurs et les modèles.
3. Configuration de Strapi :
    - Accédez au dossier **`strapi`** et exécutez **`npx create-strapi-app . --quickstart`** pour créer une nouvelle application Strapi. Utilisez le flag **`-quickstart`** pour une installation rapide avec SQLite, puis modifiez ultérieurement la configuration pour utiliser PostgreSQL.
    - Configurez les modèles, les autorisations et les paramètres de votre choix.

## **Utilisation de Docker**

Pour faciliter le développement et le déploiement, nous avons utilisé Docker et Docker Compose. Nous avons créé des fichiers **`Dockerfile`** pour l'application Next.js et l'API Strapi, ainsi qu'un fichier **`docker-compose.yml`** pour définir les services et les volumes.

### **Dockerfile pour l'application Next.js**

```docker
# Utiliser l'image officielle de Node.js avec la version 16 en tant qu'image de base
FROM node:16

# Définir le répertoire de travail
WORKDIR /app

# Copier package.json et package-lock.json
COPY package*.json ./

# Installer les dépendances
RUN npm ci

# Copier le reste des fichiers de l'application
COPY . .

# Compiler l'application pour la production
RUN npm run build

# Exposer le port pour l'application (par défaut 3000 pour Next.js)
EXPOSE 3000
# Lancer l'application en mode production
CMD ["npm", "start"]
```

### **Dockerfile pour l'API Strapi**

```docker
FROM node:16-alpine
# Installing libvips-dev for sharp Compatibility
RUN apk update && apk add --no-cache build-base gcc autoconf automake zlib-dev libpng-dev nasm bash vips-dev
ARG NODE_ENV=development
ENV NODE_ENV=${NODE_ENV}
WORKDIR /opt/
COPY ./package.json ./package-lock.json ./
ENV PATH /opt/node_modules/.bin:$PATH
RUN npm install
WORKDIR /opt/app
COPY ./ .
RUN npm run build
EXPOSE 1337
CMD ["npm", "run", "develop"]
```

### **Fichier docker-compose.yml**

```yaml
version: '3.8'

services:
  app:
    build: ./app
    ports:
      - '3000:3000'
    volumes:
      - ./app:/app # Montez le répertoire de l'application en tant que volume
      - /app/node_modules # Excluez les node_modules du volume pour éviter les conflits
    depends_on:
      - api
    environment:
      - NODE_ENV=development
    command: npm run dev # Exécutez le script de développement avec hot reload

  api:
    build: ./api
    ports:
      - '4000:4000'
    environment:
      - DATABASE_URL=postgres://postgres:password@db:5432/postgres
    depends_on:
      - db

  db:
    image: postgres:14-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=strapi
    volumes:
      - db-data:/var/lib/postgresql/data

  strapi:
    build: ./strapi
    ports:
      - '1337:1337'
    environment:
      - DATABASE_CLIENT=postgres
      - DATABASE_NAME=strapi
      - DATABASE_HOST=db
      - DATABASE_PORT=5432
      - DATABASE_USERNAME=postgres
      - DATABASE_PASSWORD=password
    volumes:
      - ./strapi:/opt/app
      - /opt/app/node_modules
    depends_on:
      - db
    command: sh -c "sleep 10 && npx strapi develop"

volumes:
  db-data:
```

### Les problèmes commencent

A ce moment impossible pour moi de connecter la bd postgres à mon container strapi

Je suis désolé d'entendre que l'erreur persiste. À ce stade, je vous recommande de vérifier si la base de données a été créée dans le conteneur **`db`**. Voici comment le faire :

1. Exécutez la commande suivante pour entrer dans le conteneur **`db`** en tant qu'utilisateur **`postgres`** :

```
docker-compose exec db psql -U postgres
```

1. Une fois que vous êtes connecté à la base de données, vérifiez si la base de données **`strapi`** a été créée en exécutant la commande suivante :

```sql
\l
```

Si la base de données **`strapi`** n'est pas répertoriée, vous pouvez la créer manuellement en exécutant les commandes suivantes :

```sql
CREATE DATABASE strapi;
GRANT ALL PRIVILEGES ON DATABASE strapi TO postgres;
```

Ensuite, quittez la console psql en tapant **`\q`** et appuyez sur Entrée.

Si vous avez dû créer la base de données manuellement, redémarrez les services en exécutant **`docker-compose down`** et **`docker-compose up -d`**. Si le problème persiste, vous pouvez vérifier les journaux du conteneur **`strapi`** pour obtenir plus d'informations sur l'erreur :

```bash
docker-compose logs strapi
```

## **Création d'un Content-Type Builder pour les articles**

Nous avons utilisé Strapi pour créer un Content-Type Builder pour nos articles. Les champs principaux sont:

- Titre
- Slug
- Auteur
- Date de publication
- Extrait
- Contenu (Markdown)
- Catégorie
- Tags
- Image principale
- SEO (à l'aide du plugin SEO)

## **Hébergement et déploiement**

Pour déployer notre application, nous avons exploré plusieurs options d'hébergement, y compris des solutions gratuites et payantes, comme Vercel, Heroku, DigitalOcean et AWS.