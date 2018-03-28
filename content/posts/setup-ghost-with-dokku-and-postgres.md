---
title: "Ghost, PostgreSQL and Dokku"
date: 2015-10-23T05:39:50-08:00
draft: false
---

There are a couple existing tutorials that cover Ghost and Dokku. Unfortunately the two I referenced were outdated and missing important details in setup decisions.

##### Versioning

Ghost: 0.4.2
Dokku: 0.2.3
Ubuntu: 14.04
PostgreSQL: latest


#### Dokku Setup

This tutorial assumes you already have a Dokku installation running. If you are not familiar with Dokku I highly recommend using Digital Oceans Dokku image to get up and running. If using Digital Ocean, be sure to start off with a 1GB droplet. I have noticed 512MB boxes do not have enough memory for installation requiring you to setup a swap file.

#### Ghost Package

- [Ghost 0.4.2](https://github.com/TryGhost/Ghost/releases/tag/0.4.2)

I had issues building the project properly for Dokku using source. The build would boot locally but not on a Dokku deploy. I am sure I missed something on the build. The current packaged release, Ghost 0.4.2, did work. Download the zipped full package, it includes all requirements that are downloaded via bower.

#### PostgreSQL

- [Jeff Utter's PostgreSQL Plugin](https://github.com/jeffutter/dokku-postgresql-plugin)

A handful of existing PostgreSQL plugins are available. I prefer Jeff's mainly because it only uses a single Postgres container and multiple databases within.

- Before installing plugins be sure to finish your Dokku installation by visiting the root address via browser. If you dont there will be permissioning errors.
- On your Dokku box go to the plugins install directory cd /var/lib/dokku/plugins
- Clone the repo git clone https://github.com/jeffutter/dokku-postgresql-plugin postgresql
- Install plugin dokku plugins-install
- Start the container dokku postgresql:start

Entire code snippet:
```
cd /var/lib/dokku/plugins  
git clone https://github.com/jeffutter/dokku-postgresql-plugin postgresql  
dokku plugins-install  
dokku postgresql:start  
```
You should now have a working PostgreSQL container!

#### Ghost

A few changes are necessary to get Ghost working properly with both PostgreSQL and Dokku.

- First we want to confirm the local download of Ghost works properly.
- In your local Ghost folder run npm install to install all dependencies.
- You should now be able to start Ghost npm start

Jeff's PostgreSQL plugin requires an app present when creating a new Database. We can deploy the current state of Ghost to create the initial Dokku app. Dokku requires a git repo for deployment.

- Create a new git repo in your local Ghost folder git init
- Add all files to the repot git add .
- Commit changes git commit -am 'init commit'
- Add Dokku remote git remote add dokku dokku@<dokku_ip>:<app_name>
- Deploy Ghost git push dokku master


Now that we have a valid Ghost app we can create the PostgreSQL DB. Its important that your DB appname matches the appname we deployed for Ghost.

- On your Dokku box create a new database dokku postgresql:create <app_name>
- Copy the connection string that is returned. It should look something like this postgres://<user>:<password>@<host>:<port>/<database>

Now that we have database lets modify Ghost to work!

- In your local Ghost folder open package.json and add ""pg"": ""latest"" to your dependencies.
- Open config.js and make the following changes in your production settings. I have included an example of what the settings should look like.
  - Update all connection settings from the connection url provided from PostgreSQL setup.
  - Update server settings to work properly with Dokku.
  - I highly recommend setting all the connection string variables as env vars. For testing it is fine to hardcode these values.

```
production: {  
    url: 'http://my-ghost-blog.com',
    mail: {},
    database: {
        client: 'postgres',
        connection: {
            host: <host>,
            user: <user>,
            password: <password>,
            database: <database>,
            port: <port>
        },
        debug: false
    },
    server: {
        host: '0.0.0.0',
        port: process.env.PORT
    }
},
```

Lastly we need to tell Dokku what to run to start the server.

- Create a file *Procfile*
- Add the following to the file to tell Dokku the specific file and method to launch the server web: node index.js


Now we need to commit all changes and deploy!

- Add new files git add .
- Commit changes git commit -am 'ghost setup'
- Deploy! git push dokku master


After deploy we have one final change. We need to tell Ghost to run with production settings. On your Dokku box create a new env var dokku config:set <app_name> NODE_ENV=production. This will set the env var and restart the app.

You should now be able to go visit your Ghost blog!