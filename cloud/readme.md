# REST API in cloud

RESTFul(Representational State Transfer) APIs can be used to collect data from edge devices such as sensors, microcontrollers (e.g., Raspberry Pi Pico W), or other IoT devices deployed at the cloud's edge.

This project implements REST API using NodeJS runtime and ExpressJS library. Once API endpoint is created for data collecting, it will be set as a service in the cloud and the API is then published by using Nginx webserver's reverse proxy.

# Prerequisites:

- [Node.js](https://nodejs.org) installed on your server and local machine(for development)
- Bash interpreter
  - Windows users can install [bash emulator](https://gitforwindows.org)
  - Linux and MacOS do have terminal and most likely suitable shell
- Headless Ubuntu 24.04 LTS Server
  - Remote access via SSH
  - Inbound HTTP and HTTPS allowed

# Create REST API project

Start REST API project creation by initializing a NodeJS project and then setting up necessary settings.

```bash
# Initialize the project and install dependencies
mkdir cloud
cd cloud
pwd # print working directory
# should point to your current dir e.g. /c/Users/username/path/to/project/cloud
mkdir src # creates source-code directory
touch src/server.mjs
# write first line code
echo "console.log('Server starting...');" >> src/server.mjs
# test if file and NodeJS is working
node src/server.mjs
# initialize cloud app as npm project
npm init -y
# write startup script in package.json or use the command below
npm pkg set scripts.start="node src/server.mjs"
# set also "module" type for EcmaScript (example uses ES imports)
npm pkg set type="module"
# test "start" script from packge.json
npm run start
# It should print: "Server starting..." 
# Install dependency for local development
npm install --save express
# it creates node_modules directory
# and adds dependency "express" to package.json
```

# Modify `src/server.mjs` to serve HTTP REST API

The example below creates Express app. It sets one endpoint for `/data` that takes query parameter `value` and logs the inserted value in server-side. It also responds to the Pico with the value it received.

```js
// "type": "module" required in package.json for following import
import express from 'express'; 
const app = express();
const port = 3000;

// Define a GET endpoint that accepts a "value" query parameter
app.get('/data', (req, res) => {
  const value = req.query.value;
    if (value) {
      console.log(`Received value from Pico W: ${value}`);
      res.send(`Value received: ${value}`);
    } else {
      res.status(400).send('Missing query parameter "value"');
    }
});

// Start the server
app.listen(port, () => {
  console.log(`Cloud API listening at http://localhost:${port}`);
});
```

Test the changes with "`npm run start`" and split terminal to send cURL request to the service.

```bash
# terminal window 1
npm run start
# terminal window 2
curl "http://127.0.0.1:3000/data?value=1234"
```

Two terminal windows are needed, because the server application holds the process open in the first one. This prevents the developer from using normal bash commands in it.

After testing with curl responding successfully ("Value received: 1234"), close the process in the first terminal window by activating it and then press <kbd>CTRL+C</kbd> 

# Setup REST API in headless Ubuntu server

Steps:

1. Login to the remote server
2. Install NodeJS (same version as locally (>=20 LTS))
3. Move REST API to the server
4. Setup REST API in server
5. Serve the REST API to the internet
6. Test API from the internet

## S1 - Login to the remote server

Commonly required login details using SSH

- `vm.public.ip.addr` - find out from the service provider
- `remote_user` - remote server username
- `keyfilepath` - e.g., `~/.ssh/keyfilename`

From now on document uses following namings in different environments:

- `local_user@pc` using local development machine
- `remote_user@vm` using remote machine in the cloud(vm - virtual machine)

Login to the remote VM with `ssh remote_user@vm.public.ip.addr -i keyfilepath`{.bash} command.

```bash
local_user@pc:~$
ssh remote_user@vm.public.ip.addr -i keyfilepath
Enter passphrase for key '/Users/local_user/.ssh/keyfilename': 
Welcome to Ubuntu...

Last login: ...
remote_user@vm:~$ 
```

Update the latest information of the available packages with `sudo apt update`{.bash} command.

```bash
remote_user@vm:~$ sudo apt update
Hit:1 http://archive.ubuntu.com/ubuntu noble InRelease
Hit:2 http://archive.ubuntu.com/ubuntu noble-updates InRelease
Hit:3 http://archive.ubuntu.com/ubuntu noble-backports InRelease
Hit:4 http://security.ubuntu.com/ubuntu noble-security InRelease
Reading package lists... Done                     
Building dependency tree... Done
Reading state information... Done
10 packages can be upgraded. Run 'apt list --upgradable' to see them.
```

## S2 - Install NodeJS v20

Check available NodeJS package version with `sudo apt list -a <package_name>` command. NodeJS package can be found from apt with name `nodejs`

```bash
remote_user@vm:~$
sudo apt list -a nodejs
Listing... Done
nodejs/noble 18.19.1+dfsg-6ubuntu5 amd64

```

`apt`-packagemanager may offer older version than what is sufficient to run the example. For convinient setup for later release, see [https://github.com/nodesource/distributions](https://github.com/nodesource/distributions) to install for example NodeJS 20.x at remote Ubuntu machine

Steps listed below as bash script (use `remote_user@vm`):

```bash
# Install curl (used later for testing)
sudo apt install -y curl
# Download nodesource setup script for NodeJS 20.x
curl -fsSL https://deb.nodesource.com/setup_20.x -o nodesource_setup.sh
# Run nodesource setup script for Node 20.x
sudo -E bash nodesource_setup.sh
# Newer NodeJS package should now be available at 
sudo apt list -a nodejs
# Install NodeJS
sudo apt install -y nodejs
# Show NodeJS version
node --version
# Should be >=20.x
```

## S3 - Move REST API to the server

Move happens from local machine to the remote machine. Pay attention to the terminal (which machine are you working on...)

Prepare `~/projects` directory for the REST API system. Use command `mkdir -p ~/projects/project_name` to create directory (flag `-p` creates parent directory if not exists).

```bash
remote_user@vm:~$ mkdir -p ~/projects/cloud_api
remote_user@vm:~$ ls -al ~/projects # check if directories exists
total 12
drwxrwxr-x 3 remote_user remote_user 4096 Sep 24 17:00 .
drwxr-x--- 8 remote_user remote_user 4096 Sep 24 17:00 ..
drwxrwxr-x 2 remote_user remote_user 4096 Sep 24 17:00 cloud-api
```

Send files from local machine to the remote by using SCP.

Default scp syntax:

`scp [options] <source> <destination>`

Options needed for transferring

- `-i <keyfilepath>` - Uses specific SSH key for login
- `-r` - Copy recursively. Required when copying directory

`<source>` and `<destination>` can be either local machine filepaths or remote machines with their filepaths.

- Local filepath
  - Relative to current path e.g., `package.json` is relative to `~/projects/edge_cloud/cloud` when it is the working directory.
  - Absolute paths are paths from the system root e.g.,
    - `/Users/username/projects/edge_cloud/cloud/package.json`
    - `~/projects/edge_cloud/cloud` same as above (tilde `~` shorthand for home directory `/Users/username`)
- Remote filepath
  - Define username@hostname:filepath
  - `remote_user@vm.public.ip.addr:~/path/to/file`

For more SCP related details, see [scp(1) - Linux man page](https://linux.die.net/man/1/scp)

Example command runs from transferring files from local to remote. Observe the filepath location.

```bash
local_user@pc:~/projects/edge_cloud$
pwd # check current working directory
/Users/local_user/projects/edge_cloud
local_user@pc:~/projects/edge_cloud$
ls -l # check current directory content
total 5
drwxr-xr-x   9 local_user  123456789   288 Sep 24 17:26 cloud
drwxr-xr-x   5 local_user  123456789   160 Sep 24 16:59 edge_v2
-rw-r--r--   1 local_user  123456789  2932 Sep 24 18:20 readme.md
local_user@pc:~/projects/edge_cloud$
cd cloud # change to cloud project directory
local_user@pc:~/projects/edge_cloud/cloud$
pwd # check current working directory again
/Users/local_user/projects/edge_cloud/cloud
local_user@pc:~/projects/edge_cloud/cloud$
ls -l
total 96
drwxr-xr-x  67 local_user  123456789   2144 Sep 24 01:45 node_modules
-rw-r--r--   1 local_user  123456789  25915 Sep 24 01:45 package-lock.json
-rw-r--r--   1 local_user  123456789    355 Sep 24 20:17 package.json
drwxr-xr-x   3 local_user  123456789     96 Sep 24 20:16 src
local_user@pc:~/projects/edge_cloud/cloud$
scp -i ~/.ssh/keyfilename package.json remote_user@vm.public.ip.addr:~/projects/cloud_api
Enter passphrase for key '/Users/local_user/.ssh/keyfilename': 
package.json                                               100%  355     6.4KB/s   00:00    
local_user@pc:~/projects/edge_cloud/cloud$
scp -i ~/.ssh/keyfilename -r src remote_user@vm.public.ip.addr:~/projects/cloud_api
Enter passphrase for key '/Users/local_user/.ssh/keyfilename': 
server.mjs                                                 100%  561    10.5KB/s   00:00 
```

Confirm that the files are in remote machine (VM)

```bash
remote_user@vm:~$ ls -l ~/projects/cloud_api
-rw-r--r--  1 remote_user remote_user   355 Sep 24 17:26 package.json
drwxr-xr-x  2 remote_user remote_user  4096 Sep 24 12:14 src
remote_user@vm:~$ ls -l ~/projects/cloud_api/src
total 4
-rw-r--r-- 1 remote_user remote_user 561 Sep 24 17:32 server.mjs
```

## S4 - Setup REST API in server

Procedures to setup REST API in a server:

1. Install the REST API Dependencies
2. Start the REST API
3. Test the REST API

### S4P1 - Install the REST API Dependencies

Run the `npm install` command in the directory containing `package.json`.

```bash
remote_user@vm:~$ cd ~/projects/cloud_api
remote_user@vm:~/projects/cloud_api$ npm install

added 65 packages, and audited 66 packages in 961ms

13 packages are looking for funding
  run `npm fund` for details

found 0 vulnerabilities
```

### S5P2 - Start the REST API

Start cloud api in remote machine (terminal window 1)

```bash
remote_user@vm:~$ cd ~/projects/cloud_api
remote_user@vm:~/projects/cloud_api$ npm run start

> cloud-api@1.0.0 start
> node src/server.mjs

Server starting...
Cloud API listening at http://localhost:3000

```

### S5P3 - Test the REST API

Open up a new remote connection to the remote machine (terminal window 2). Then test the REST API inside the remote machine using cURL. This will help to ensure that the REST API is working properly within the server.

Open up secondary terminal window(2)

1. Connect with ssh `ssh remote_user@vm.public.ip.addr -i keyfilepath`
2. Test running REST API with cURL `curl "http://127.0.0.1:3000/data?value=1234"`

Example below:

```bash
local_user@pc:~$
ssh remote_user@ip.addr.to.vm -i ~/.ssh/keyfilename
Enter passphrase for key '/Users/local_user/.ssh/keyfilename': 
Welcome to Ubuntu 24.04.1 LTS
...
remote_user@vm:~$ curl "http://127.0.0.1:3000/data?value=1234"
Value received: 1234remote_user@vm:~$ 
```

### S5 - Serve REST API to the internet

This step expects that the inbound HTTP(80) and HTTPS(443) traffic is routed to the server. Most of the time this can be set when allocating the virtual machine from a service provider or the inbound traffic is not blocked at all by default.

This step also uses Nginx as webserver. If you don't have a webserver on the remote machine yet, install nginx by running command:

```bash
remote_user@vm:~$ sudo apt install -y nginx
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
...
```

By default the webserver starts automatically as a service. Test if the nginx test site is visible by using URL bar `http://public.vm.ip.addr`.

Some commands often run when setting up installed service:

```bash
sudo systemctl status nginx.service # check status
sudo systemctl enable nginx.service # start service on boot
sudo systemctl start nginx.service # restart service
sudo systemctl restart nginx.service # restart service
```

Now modify the Nginx's default site configuration at `/etc/nginx/sites-enabled/default` This requires sudo privileges. After modification, save changes.

Nano editor sequence to save changes:

1. <kbd>CTRL+X</kbd>
2. <kbd>Y</kbd>
3. <kbd>Enter</kbd>

Test the configuration and apply if syntax ok. See the required modifications below the example commands run.

**Example commands run:**

```bash
remote_user@vm:~$ sudo nano /etc/nginx/sites-enabled/default
# add the reverse proxy for cloud API (port 3000)
remote_user@vm:~$ sudo nginx -t # test nginx config
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
remote_user@vm:~$ sudo nginx -s reload # nginx way to restart (service specific)
2024/09/24 18:28:35 [notice] 155693#155693: signal process started
remote_user@vm:~$ sudo systemctl restart nginx # systemd way to restart (system specific)
remote_user@vm:~$ sudo systemctl status nginx # show service status
```

Add only the content between "`MODIFY`" part to the configuration at the specified location.

```nginx
server {
        # ...
        server_name _; # add reverse proxy below this line
        # MODIFY - reverse proxy block starts
        location /api/v1/ {
                proxy_pass http://127.0.0.1:3000/;
        }
        # MODIFY - reverse proxy block ends
        location / { # default configuration continues
        # ...
}
```

After the configuration is applied as in the above described, the REST API should be served to the internet

### S6 - Test the API from the internet

Use browser to test

`http://your.public.ip.addr/api/v1/data?value=12345`

If the browser responds with the specified value, then you've got your API online

## Run the REST API 24/7

Set the REST API to run 24/7 on the server by defining it as a service via Systemd

Example service file (Save to `/etc/systemd/system/cloud-api.service`)

```service
[Unit]
Description=cloud-api
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/bin/node /home/username/projects/cloud_api/src/server.mjs
Restart=on-failure

[Install]
WantedBy=multi-user.target

```

The service file can be created with Nano editor e.g.,

```bash
remote_user@vm:~$ sudo nano /etc/systemd/system/cloud-api.service
```

Command `realpath ~/projects/cloud_api/src/server.mjs` reveals full filepath(absolute) to the file to start.

Finally reload systemd files, then start, enable(starts service on boot) and check the service.

```bash
remote_user@vm:~$ sudo systemctl daemon-reload
remote_user@vm:~$ sudo systemctl start cloud-api.service
remote_user@vm:~$ sudo systemctl enable cloud-api.service
Created symlink /etc/systemd/system/multi-user.target.wants/cloud-api.service → /etc/systemd/system/cloud-api.service.
remote_user@vm:~$ sudo systemctl status cloud-api.service
● cloud-api.service - cloud-api
     Loaded: loaded (/etc/systemd/system/cloud-api.service; enabled; preset: enabled)
     Active: active (running) since Tue 2024-09-24 18:54:21 UTC; 36s ago
   Main PID: 155934 (node)
      Tasks: 11 (limit: 1064)
     Memory: 10.6M (peak: 15.6M)
        CPU: 198ms
     CGroup: /system.slice/cloud-api.service
             └─155934 /usr/bin/node /home/remote_user/projects/cloud_api/src/server.mjs

Sep 24 18:54:21 diotp-sys1 systemd[1]: Started cloud-api.service - cloud-api.
Sep 24 18:54:21 diotp-sys1 node[155934]: Server starting...
Sep 24 18:54:21 diotp-sys1 node[155934]: Cloud API listening at http://localhost:3000
```

# Conclusion

Now that the REST API is active, try the `edge_v2` example and see if you can send data to the cloud!

