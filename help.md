First connect to the VPS from PowerShell (run as admin):

```bash
ssh ubuntu@YOUR_VPS_IP
```

Update the system and install basic tools:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install git curl nano -y
```

Install Node.js (needed for your server, Bare proxy, and Discord bot):

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install nodejs -y
```

Check it installed:

```bash
node -v
npm -v
```

---

Make the `/srv` folder where the project will live:

```bash
sudo mkdir /srv
sudo chown ubuntu:ubuntu /srv
cd /srv
```

Clone your repo:

```bash
git clone https://github.com/YOURNAME/YOURREPO.git
cd YOURREPO
npm install
```

---

Install the web server Caddy:

```bash
sudo apt install caddy -y
```

Start and enable it:

```bash
sudo systemctl start caddy
sudo systemctl enable caddy
```

---

Login to your domain registrar Spaceship and create DNS records:

```
Type: A
Name: @
Value: YOUR_VPS_IP
```

Optional wildcard:

```
Type: A
Name: *
Value: YOUR_VPS_IP
```

If you're using custom domains from FreeDNS, create A records there pointing to the same VPS IP.

---

Install the proxy server TompHTTP Bare Server globally:

```bash
npm install -g @tomphttp/bare-server-node
```

Bare will run on port **8080** and will be accessed at:

```
yourdomain.com/bare/
```

---

Inside your repo:

```bash
nano server.js
```

Example simple server:

```javascript
const http = require("http");

const PORT = 3000;

http.createServer((req,res)=>{
 res.writeHead(200,{"Content-Type":"text/plain"});
 res.end("Server running");
}).listen(PORT, ()=>console.log("Server running on port "+PORT));
```

---

Edit the config:

```bash
sudo nano /etc/caddy/Caddyfile
```

Example setup:

```
yourdomain.com {

 handle_path /bare/* {
  reverse_proxy localhost:8080
 }

 reverse_proxy localhost:3000

}
```

Meaning:

```
yourdomain.com → Node server
yourdomain.com/bare/ → Bare proxy
```

Reload the config:

```bash
sudo systemctl reload caddy
```

Caddy automatically generates HTTPS certificates.

---

Create the service:

```bash
sudo nano /etc/systemd/system/nodeapp.service
```

Paste:

```
[Unit]
Description=Node Web Server
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/srv/YOURREPO
ExecStart=/usr/bin/node server.js
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

Enable and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable nodeapp
sudo systemctl start nodeapp
```

Check status:

```bash
sudo systemctl status nodeapp
```

---

Create service:

```bash
sudo nano /etc/systemd/system/bare.service
```

Paste:

```
[Unit]
Description=Bare Proxy Server
After=network.target

[Service]
Type=simple
User=ubuntu
ExecStart=/usr/bin/npx bare-server-node --port 8080
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

Enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable bare
sudo systemctl start bare
```

---

Install the library Discord.js:

```bash
npm install discord.js
```

Create the bot file:

```bash
nano bot.js
```

Basic bot with update command:

```javascript
const { Client, GatewayIntentBits } = require("discord.js");
const { exec } = require("child_process");

const client = new Client({
 intents:[
  GatewayIntentBits.Guilds,
  GatewayIntentBits.GuildMessages,
  GatewayIntentBits.MessageContent
 ]
});

client.on("messageCreate", msg=>{

 if(msg.content==="!update"){

  exec("cd /srv/YOURREPO && git pull", ()=>{

   exec("sudo systemctl restart nodeapp");
   exec("sudo systemctl reload caddy");

   msg.reply("server updated");
  });

 }

});

client.login("BOT_TOKEN");
```

---

Create service:

```bash
sudo nano /etc/systemd/system/discordbot.service
```

Paste:

```
[Unit]
Description=Discord Management Bot
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/srv/YOURREPO
ExecStart=/usr/bin/node bot.js
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

Start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable discordbot
sudo systemctl start discordbot
```

---

Example command `!adddomain` could append to the Caddyfile:

```
exampledomain.com {
 reverse_proxy localhost:3000
}
```

Then reload Caddy:

```bash
sudo systemctl reload caddy
```
