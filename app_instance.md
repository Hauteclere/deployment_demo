Yesterday, we got our database live and were able to access it from our locally hosted app. Today we are going to push our app to the cloud and get it hooked up so that other people can access it.

## Step A: Create the EC2

We set up a security group for our instance yesterday; now we need to create the instance itself. Make sure you assign it the group we created for it. Remember that the correct group is the one with access for incoming requests on ports `80` and `443`, (NOT the group that we used for our database, which has is available on port `5432`).

Once again, you'll need to pick a key-pair for SSh purposes. I recommend using the one you generated yesterday, as it will save fiddling around with a new key.

Next, we'll set up that same SSh quality-of-life feature we did for the database EC2. Open `~/.ssh/config`  in an editor and add the following code:

```
Host flask-app
   Hostname IP_OF_INSTANCE_HERE
   User ubuntu
   IdentityFile ~/.ssh/NAME_OF_PEM_FILE.pem
   IdentitiesOnly yes
```

You should be now able to SSh into it with the command:

```
ssh flask-app
```

---

## Step B: Setting up the environment

Now we need to install the important programs we need to support this app.

```
sudo apt update

sudo apt install python3-pip

sudo apt install python3-virtualenv

sudo apt install nginx
```


(Nginx is going to be our reverse-proxy, remember! Check out the architecture diagram on the first slide in this lesson if you're confused by it.) 

You should also check you have Git installed, just to be safe:

```
git --version
```


Next we can clone down the repo from github:

```
git clone https://github.com/Oliver-CoderAcademy/streets_app.git
```


Jump into the app directory and create a virtualenv:

```
cd streets_app

virtualenv venv
```


Activate the venv, and install the requirements:

```
source venv/bin/activate

pip install -r requirements.txt
```

(You may note that the Gunicorn library is in the list of requirements. If you've been working with Flask's built-in development server up until now, you won't have had to have Gunicorn installed, so you might not have added it to your requirements yet. Make sure that it's included. You could always just run a quick `pip install` of it now, but if you do that then your `requirements.txt` will also be wrong the next time you install this app somewhere...) 



And add a `streets_app/.env` file for our environment variables!

```
DB_USER = "ubuntu"
DB_PASS = "your_password_here"
DB_NAME = "your_db_name_here"
DB_DOMAIN = "your_db_IP_here:5432"
SECRET_KEY = "some secret string"
```

---

## Step C: Setting up Gunicorn

We are going to use `systemd` to run our Gunicorn server. This means that running our app will be incorporated into the automatic processes of the instance, so we shouldn't have to worry as much about manually starting it up. 

To do this, we create a file with the `.service` suffix, in the `/etc/systemd/system` directory, like so:

```
sudo nano /etc/systemd/system/streets_app.service
```

Here are the contents this file needs:

```
[Unit]
Description=Gunicorn instance for a simple flask app
After=network.target
[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/streets_app/streets_app
ExecStart=/home/ubuntu/streets_app/venv/bin/gunicorn -b localhost:8000 "main:create_app()"
EnvironmentFile=/home/ubuntu/streets_app/.env
Restart=always
[Install]
WantedBy=multi-user.target
```

This file could use some discussion - I'll talk through what is happening here during class.



Finally, we can get that service running!

```
sudo systemctl daemon-reload

sudo systemctl start streets_app

sudo systemctl enable streets_app
```


You can check it worked with:

```
curl localhost:8000
```

---

## Step D: Nginx time!

We can start up Nginx with 

```
sudo systemctl start nginx

sudo systemctl enable nginx
```

But that will just set it to serve the default blank landing page. To get it serving our app, we need to edit its default behaviour:

```
sudo nano /etc/nginx/sites-available/default
```

This file has a ton of comments in it. Here's what we need to edit it to:

```
upstream streets_app {
        server 127.0.0.1:8000;
}


server {
        listen 80 default_server;
        listen [::]:80 default_server;
        root /var/www/html;

        index index.html index.htm index.nginx-debian.html;

        server_name _;

        location / {
                proxy_pass http://streets_app;
        }
}
```

We have:

1. added the section with upstream at the start, and

1. edited the section beginning with `location /`

Feel free to leave all those comments in there - I've only cut them out for ease of reading here.

Now we just need to restart Nginx:

```
sudo systemctl restart nginx
```

Believe it or not, our app should be working!

