These instructions make the following assumptions:
- You've just finished setting up a Linux server (say, Ubuntu 16.04 64-bit) and have installed Docker on it.

- You've configured your security groups to allow for incoming SSH connections from your local IP.

- You've configured a domain name (or subdomain) to point to your server's IP address.

- You've configured the DNS to enable HTTPS for your domain (say, using Cloudflare).

### Getting started

1. SSH into your new server:

   ``` bash
   $ ssh -l {user} {IP to the server}
   ```

2. Update your system:

   ``` bash
   $ sudo apt-get update
   $ sudo apt-get upgrade
   ```

3. Install git

   ``` bash
   $ sudo apt-get update
   $ sudo apt-get install -y git
   ```

4. Make sure you are in your home directory and clone the Standard File [ruby-server](https://github.com/standardfile/ruby-server) project:

   ``` bash
   $ cd ~
   $ git clone https://github.com/standardfile/ruby-server.git
   $ cd ruby-server
   ```

5. Create .env.{app|db}.production files in the project's root directory. Add environment variables (see Environment variables for full listing):

   ``` bash
   $ cd $PROJECT_ROOT/docker/environments/
   $ cp .env.app.producion.template .env.app.production
   $ vim .env.app.production
   ```

   Insert:

   ```
   RAILS_ENV=production
   SECRET_KEY_BASE=use "bundle exec rake secret"
   RAILS_SERVE_STATIC_FILES=true

   DB_HOST=localhost
   DB_PORT=3306
   DB_DATABASE=items
   DB_USERNAME=root
   DB_PASSWORD=

   SALT_PSEUDO_NONCE=use "bundle exec rake secret"
   ```

   ``` bash
   $ cd $PROJECT_ROOT/docker/environments/
   $ cp .env.db.producion.template .env.db.production
   $ vim .env.db.production
   ```

   Insert:

   ```
   MYSQL_ROOT_PASSWORD=
   ```

6. Build the services, without starting them:
   ``` bash
   $ docker-compose build
   ```

6. Run the app service to compile the assets:
   ``` bash
   $ docker-compose -f docker-compose.yml -f docker-compose.production.yml up app
   $ docker ps
   $ docker exec -it {container ID/name of the app service} /bin/bash
   app> bundle exec rake assets:precompile
   ```

   At this point the precompiled assets are stored in the `public/`
   folder of the host. Nginx container will mount the folder as volume
   and get the assets.

   ``` bash
   $ docker-compose down
   ```

6. Start the services:

   ``` bash
   $ docker-compose -f docker-compose.yml -f docker-compose.production.yml up -d
   ```

7. Login to the `app` service to initialize project:
   ``` bash
   $ docker ps
   $ docker exec -it {container ID/name of the app service} /bin/bash
   app> bundle exec rake db:create db:migrate
   ```

8. Access the server locally:
  ``` bash
  $ curl {domain name}
  <!doctype html>
  <html>
    ...
    <body>
      <h1> Hi! You're not supposed to be here. </h1>

      <p> You might be looking for the <a href="https://app.standardnotes.org"> Standard Notes Web App</a> or the main <a href="https://standardnotes.org"> Standard Notes Website</a>. </p>

    </body>
  </html>
  ```

9. You're done!

## Using your new server
You can immediately start using your new server by using the Standard Notes app at https://app.standardnotes.org.

In the account menu, enter the address of your new server and press Change Server:

![sn-account-menu](http://imgur.com/Pre6ffL.png)

Then, register for a new account, and begin using your private new secure Standard File server!
