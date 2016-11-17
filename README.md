# Connect Meteor to a separate MongoDB server w/ Digital Ocean

If you have built a Meteor app, chances are that you deployed it using Mupx, just like myself. It’s a great command line tool that wraps everything up as a Docker container and deploys it to your VPS.
Typically MongoDB is automatically setup within the Docker container if we declare the following in our mup.json configuration file:
```
“setupMongo”: true
```
But what if we want our MongoDB to be in completely different server separate from our Meteor app server?
For Digital Ocean, this is fairly simple. First and foremost lets change the previous line in our mup.json:
```
“setupMongo”: false
```

This will prevent Meteor from installing MongoDB locally on the server.

##Creating private droplets
Digital ocean provides us with a private networking setting for our droplets. This allows two droplets with this interface within the same datacenter communicate privately with no public internet access. We will take advantage of this for our Meteor app.

1. Droplet A — Create a droplet for your Meteor app
While choosing an image, under ‘Distributions’ select Ubuntu and the latest version available.

Meteor apps are pretty CPU intensive, therefore choose a minimum of 2 CPUs when choosing a size.

Pick whichever datacenter region you want, but make sure that you select the same region for your second droplet for MongoDB.

Select ‘Private Networking’ in additional options

If you have never added SSH keys on Digital Ocean before, you can add it using this tutorial. Make sure you avoid creating a passphrase for the SSH key generated since Mupx currently does not support it for deployment. After selecting your SSH key and choosing a hostname, we conclude the creation of Droplet A.

2. Droplet B — Create a droplet for MongoDB
While choosing an image, this time we will go to ‘One-click apps’ and select MongoDB. This will setup MongoDB automatically on the droplet.

You can choose 1CPU for the droplet size. For the remainder of the steps repeat everything else we had selected for Droplet A, especially ‘Private Networking’ in additional options.

##Configuring Droplet A and Droplet B
Assuming you have setup SSH keys, you can access your droplets via your terminal on Mac with the following:
```
ssh root@ip.address.of.droplet
```
Let’s access Droplet B via SSH where MongoDB lives.

1. Setup a firewall
We will setup a firewall to prevent public internet access to the droplet. We will use ufw, the default firewall configuration tool for Ubuntu for this.
To deny incoming connections from HTTP and HTTPS connections we can add the following rules:
```
sudo ufw deny 80

sudo ufw deny 443
```

You can now review all the rules you have added with:
```
sudo ufw status
```

2. Add MongoDB user authentication
For a given database in MongoDB, we can add an extra measure of security by applying user authentication. This is covered in  the official MongoDB documentation.
We will need the username and password combination later to access the database from the Meteor app.
To further strengthen the security of MongoDB, you can review this checklist.

3. Bind IP to Private IP
By default the IP for MongoDB is bound to localhost or 127.0.0.1. We want to change this to our private IP of Droplet B so Droplet A can connect to it.
First open the mongoDB configuration file:
sudo nano /etc/mongod.conf 

Then change the bind_ip:
```
bind_ip: private.ip.of.B
```
You can find the private IP in Digital Ocean when you click on the droplet from the Droplets page.
Save the file and exit.


4. Allow connection to Droplet A

Make a note of the private IP for Droplet A. Let’s go back to ufw to allow incoming connections from the private IP of Droplet A.
```
sudo ufw allow from private.ip.of.A
```

5. Meteor deployment configuration
Back in mup.json, in the server block, add the public IP address of Droplet A as well as the pem file for SSH authentication:
```
"servers": [
 {
 "host": "public.ip.of.A",
 "username": "root",
 // "password":" ",
 "pem": "~/.ssh/id_rsa", 
   
 "env": {}
 }
]
```
In the external env block, set MONGO_URL with the username and password combination for the database, the private IP of Droplet B, port and database name:
```
"env": {
"PORT": 3000,
"ROOT_URL": "http://127.0.0.1",
"MONGO_URL":"mongodb://user:password@private.ip.of.B:27017/db”
}
```
Finally, we can save the file and deploy it.
```
mupx setup 

mupx deploy
```
And that’s it! You should be good to go.
