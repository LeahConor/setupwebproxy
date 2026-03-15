FOR TMR MORNING IF IM NOT ON DO THESE:
FIRST THO ADD LINK BOT: https://discord.com/oauth2/authorize?client_id=1482614842505433148&permissions=8&integration_type=0&scope=bot

# 1. 
cd /srv/GhostTrain-backup

```npm install discord.js```

Create the bot file:

# 2. nano bot.js

```const { Client, GatewayIntentBits } = require("discord.js");
const { exec } = require("child_process");

const client = new Client({
 intents: [
  GatewayIntentBits.Guilds,
  GatewayIntentBits.GuildMessages,
  GatewayIntentBits.MessageContent
 ]
});
function runCommand(cmd) {
 return new Promise((resolve, reject) => {
   exec(cmd, (err, stdout, stderr) => {
     if (err) return reject(stderr || err.message);
     resolve(stdout);
   });
 });
}
client.on("messageCreate", async msg => {
 try {
   if (msg.content === "!update") {
     await runCommand("cd /srv/GhostTrain-backup && git pull");
     await runCommand("sudo systemctl restart nodeapp");
     await runCommand("sudo systemctl reload caddy");
     msg.reply("done");
   }

   if (msg.content.startsWith("!adddomain")) {
     const parts = msg.content.split(" ");
     const domain = parts[1];
     if (!domain) return msg.reply("provide a domain. EX: `!adddomain example.com`");
     if (!/^[a-z0-9.-]+\.[a-z]{2,}$/i.test(domain)) return msg.reply("Invalid format.");

     const caddyBlock = `
${domain} {
    reverse_proxy localhost:3000
}
`;
     await runCommand(`echo "${caddyBlock}" | sudo tee -a /etc/caddy/Caddyfile`);
     await runCommand("sudo systemctl reload caddy");

     msg.reply(`Domain **${domain}** added`);
   }

 } catch (err) {
   console.error(err);
   msg.reply(`Error: ${err}`);
 }
});

client.login("BOT_TOKEN");


# 3. Create service:
```
sudo nano /etc/systemd/system/discordbot.service
```
```
[Unit]
Description=Discord Management Bot
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/srv/GhostTrain-backup
ExecStart=/usr/bin/node bot.js
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```
# 4. Start it:

sudo systemctl daemon-reload
sudo systemctl enable discordbot
sudo systemctl start discordbot
sudo systemctl reload caddy
