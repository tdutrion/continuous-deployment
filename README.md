Continuous Deployment
---------------------

This project is an example of continuous deployment of a Symfony 4 project.

 * Symfony 4
 * PHP 7.2 and Apache (mod_php)
 * MYSQL 5.7
 * Doctrine ORM

The Symfony project is slightly modified with Controller As A Service, Zend Template Renderer, PSR-7 requests and
responses, PSR-15 request handlers.

## Local docker environment

The local docker environment uses the code as a volume, and has a mysql and a minio server for persistance.

## First deployment

We will first deploy the project locally on the local machine using git.

We are assuming the project is currently properly versioned. We will need to create a repository that will act as a
remote for us, then edit its hooks to trigger actions when receiving code, and finally set a symlink to handle the
webserver configuration.

#### Create the folder structure

Assuming `./` as your current directory, empty for now.

Clone this repository in the `./project` folder (`git clone git@github.com:tdutrion/continuous-deployment.git project`).

Then create an empty repository using `mkdir remote && cd remote && git init --bare`.

Get back to your initial directory (`cd ..`).

Finally, let's create our deployment folders (`mkdir -p deployment/{blue,green}`).

#### Configure git

Copy the [post-receive hook](https://www.digitalocean.com/community/tutorials/how-to-use-git-hooks-to-automate-development-and-deployment-tasks) sample and edit it:

```bash
touch remote/hooks/post-receive && chmod +x remote/hooks/post-receive
```

```bash
#!/bin/bash

while read oldrev newrev ref
do
    if [[ $ref =~ .*/master$ ]];
    then
        cd ..
        echo "Master ref received.  Deploying master branch to production..."
        deployment_dir=$(readlink -- "deployment/current")
        echo "Currently deployed: $deployment_dir"
        temp_dir="/tmp/$(uuidgen)";
        mkdir $temp_dir;
        if [ "$deployment_dir" != "./blue/public" ]; then
            echo "building for blue";
            if [ -d $temp_dir ]; then
                rm -r $temp_dir;
            fi
            git clone remote $temp_dir;
            APP_ENV=prod composer install --working-dir=$temp_dir --prefer-dist --no-dev --no-suggest --optimize-autoloader --classmap-authoritative;

            # Check whether everything went as planned.

            if [ -d deployment/blue ]; then
                rm -r deployment/blue;
            fi
            cp -R $temp_dir deployment/blue
            if [ -L deployment/current ]; then
                rm deployment/current;
            fi
            ln -s ./blue/public deployment/current;
        else
            echo "building for green";
            if [ -d $temp_dir ]; then
                rm -r $temp_dir;
            fi
            git clone remote $temp_dir;
            APP_ENV=prod composer install --working-dir=$temp_dir --prefer-dist --no-dev --no-suggest --optimize-autoloader --classmap-authoritative;

            # Check whether everything went as planned.

            if [ -d deployment/green ]; then
                rm -r deployment/green;
            fi
            cp -R $temp_dir deployment/green
            if [ -L deployment/current ]; then
                rm deployment/current;
            fi
            ln -s ./green/public deployment/current;
        fi
        echo "SetEnv APP_ENV prod" > deployment/current/.htaccess;
        rm -rf $temp_dir;
    else
        echo "Ref $ref successfully received.  Doing nothing: only the master branch may be deployed on this server."
    fi
done
```

This script can obviously be optimised but you get the idea.

Make sure it is executable (`chmod +x remote/hooks/post-receive`).


Then make sure we know which one of green or blue is the current deployment. Because we're on a single server,
we will use a symlink. The same concept could be applied using DNS resolution or a reverse proxy configuration.

```bash
ln -s ./blue/public deployment/current
```

After you need to go in project folder
```bash
cd ./project
```

Now you can change the remote of project
```bash
git remote rm origin
git remote add origin ../remote
git push origin master
```

Last thing we need to do here is to set a webserver with a document root pointing on the public directory of the symlink.

```bash
mkdir env
touch env/Dockerfile
```

Add the following content to the Dockerfile:

```dockerfile
FROM php:7.2-apache

RUN docker-php-ext-install pdo pdo_mysql \
 && a2enmod rewrite \
 && sed -i 's!/var/www/html!/var/www/html/deployment/current!g' /etc/apache2/sites-available/*.conf

WORKDIR /var/www/html
```

```bash
docker build -t continuous-deployment/server-demo env

UNIX
docker run --rm -v $(pwd):/var/www/html:cached -p 80:80 continuous-deployment/server-demo

WINDOWS
docker run --rm -v /$(pwd):/var/www/html:cached -p 80:80 continuous-deployment/server-demo

```

You need to be careful that port 80 is not used already, or change it in the command.

You can go on http://127.0.0.1 and you should see one page with Hello World

Now you can try to edit content of message
```bash
vim project/src/Action/Home.php
```

Now you can do the same things what you did before, and you will see the page change without downtime (you may need to clear the webbrowser http cache)

```bash
git status
git add .
git commit -m "second commit"
git push origin master
```

## Build Docker

Launch a docker registry:

```bash
docker run -d -p 6000:5000 --name registry registry:2
```

Then try to push your existing image:

```bash
docker tag continuous-deployment/server-demo localhost:6000/continuous-deployment/server-demo
docker push localhost:6000/continuous-deployment/server-demo
```

When you then try to run it, an error will occur as we haven't added the code yet. Add a new Dockerfile in `./Dockerfile`:

```dockerfile
FROM php:7.2-apache

RUN docker-php-ext-install pdo pdo_mysql \
 && a2enmod rewrite \
 && sed -i 's!/var/www/html!/var/www/html/deployment/current!g' /etc/apache2/sites-available/*.conf

ADD . /var/www/html

WORKDIR /var/www/html
```

In order not to add the whole context, add a `./.dockerignore`:

```
project/
remote/
env/
```

```bash
docker build -t continuous-deployment/server-demo .
docker run --rm -p 80:80 continuous-deployment/server-demo
```
