# Google Drive Treasure Box

> Not just the fastest copy tool for google drive [comparison with other tools] (./compare.md)

## Function Introduction
This tool currently supports the following features:
-Statistics of any (you have the relevant permissions, the same below, no longer repeat) the file information of the directory, and supports export in various forms (html, table, json).
Support interrupt recovery, and the statistics of the directory (including all its descendants directory) information will be recorded in the local database file (gdurl.sqlite)
Please enter `./count -h` on the command line in the project directory to view the help

-Copy all files in any directory to the directory you specify, also support interrupt recovery.
Support filtering according to file size, you can enter `./copy -h` to view the help

-Deduplicate any directory, delete files with the same md5 value in the same directory (only one is kept), and delete empty directories.
Command line input `./dedupe -h` View usage help

-After completing the relevant configuration in config.js, you can deploy this project on the server (which can normally access Google services), providing http api file statistics interface

-Support telegram bot, after configuration, the above functions can be operated by bot

## demo
[https://drive.google.com/drive/folders/124pjM5LggSuwI1n40bcD5tQ13wS0M6wg](https://drive.google.com/drive/folders/124pjM5LggSuwI1n40bcD5tQ13wS0M6wg)

## Environment configuration
This tool needs to install nodejs. For client installation, please visit [https://nodejs.org/zh-cn/download/](https://nodejs.org/zh-cn/download/), server installation can refer to [https ://github.com/nodesource/distributions/blob/master/README.md#debinstall](https://github.com/nodesource/distributions/blob/master/README.md#debinstall)

If your network environment cannot access Google services normally, you need to configure some on the command line first: (skip this section if you can access it normally)
```
http_proxy="YOUR_PROXY_URL" && https_proxy=$http_proxy && HTTP_PROXY=$http_proxy && HTTPS_PROXY=$http_proxy
```
Please replace `YOUR_PROXY_URL` with your own proxy address

## Depends on installation
-Command line execution `git clone https://github.com/iwestlin/gd-utils && cd gd-utils` clone and switch to this project folder
-Execute `npm i` to install dependencies, some dependencies may require agent environment to download, so the previous configuration is required

If an error occurs during installation, please switch the nodejs version to v12 and try again. If there is a message like "Error: not found: make" in the error message, it means your command line environment is missing the make command, you can refer to [here](https://askubuntu.com/questions/192645/make-command- not-found) or directly google search `Make Command Not Found`

After the dependency installation is complete, there will be an additional `node_modules` directory in the project folder. Please do not delete it. Then proceed to the next configuration.

## Service Account configuration
It is strongly recommended to use a service account (hereinafter referred to as SA), please refer to [https://gsuitems.com/index.php/archives/13/](https://gsuitems.com/index.php/archives/13/ #%E6%AD%A5%E9%AA%A42%E7%94%9F%E6%88%90serviceaccounts)
After obtaining the SA json file, please copy it to the `sa` directory

After configuring the SA, if you do not need to operate the files in the personal disk, you can skip the [personal account configuration] section, and when executing the command, remember to bring the `-S` parameter to tell the program to use SA authorization to operate .

## Personal account configuration
-Run `rclone config file` on the command line to find the path of rclone's configuration file
-Open the configuration file `rclone.conf`, find the three variables `client_id`, `client_secret` and `refresh_token`, and fill them in `config.js` under this project respectively, you need to pay attention to these three values Must be wrapped in pairs of English quotation marks, and the quotation marks end with a comma, which is required to comply with JavaScript [Object Syntax](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/ Operators/Object_initializer)

If you have not configured rclone, you can search for `rclone google drive tutorial` to complete the relevant configuration.

If your `rclone.conf` does not have `client_id` and `client_secret`, it means that when you configure rclone, you use rclone’s own client_id by default, even rclone [does not recommend it] (https://github.com /rclone/rclone/blob/8d55367a6a2f47a1be7e360a872bd7e56f4353df/docs/content/drive.md#making-your-own-client_id), because everyone shares its interface call limit, the limit may be triggered during peak usage periods.

You can refer to these two articles to get your own clinet_id: [Cloudbox/wiki/Google-Drive-API-Client-ID-and-Client-Secret](https://github.com/Cloudbox/Cloudbox/wiki/Google-Drive -API-Client-ID-and-Client-Secret) and [https://p3terx.com/archives/goindex-google-drive-directory-index.html#toc_2](https://p3terx.com/archives/ goindex-google-drive-directory-index.html#toc_2)

After obtaining client_id and client_secret, execute `rclone config` again to create a new remote. **In the configuration process, you must fill in your newly acquired clinet_id and client_secret**, which can be in `rclone.conf` See the newly acquired `refresh_token`. **Note that the previous refrest_token** cannot be used because it corresponds to the client_id that comes with rclone

After the parameters are configured, execute `node check.js` on the command line. If the command returns the data of the root directory of your Google hard disk, it means that the configuration is successful and you can start using the tool.

## Bot configuration
If you want to use the telegram bot function, further configuration is required.

First get the token of the bot according to the instructions at [https://core.telegram.org/bots#6-botfather](https://core.telegram.org/bots#6-botfather), then fill in config.js The `tg_token` variable in.

Next, you need to deploy the code to the server.
If you configured it on the server from the beginning, you can directly execute `npm i pm2 -g`

If you have operated locally before, please repeat it on the server again. After configuring the relevant parameters, execute `npm i pm2 -g` to install the process daemon pm2

After installing pm2, execute `pm2 start server.js`. After the code is run, it will listen to `23333` port on the server.

*If you don’t want to use nginx, you can change `23333` in `server.js` to `80` to listen directly to port 80 (may require root permission)*

Next, you can start a web service through nginx or other tools. Example nginx configuration:
```
server {
  listen 80;
  server_name your.server.name;

  location / {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_pass http://127.0.0.1:23333/;
  }
}
```
After configuring nginx, you can set another layer of cloudflare, please search for specific tutorials.

Check if the website is successfully deployed, you can execute it from the command line (please replace YOUR_WEBSITE_URL with your URL)
```
curl'YOUR_WEBSITE_URL/api/gdurl/count?fid=124pjM5LggSuwI1n40bcD5tQ13wS0M6wg'
```
![](./static/count.png)

If such file statistics are returned, the deployment is successful.

Finally, execute on the command line (please replace [YOUR_WEBSITE] and [YOUR_BOT_TOKEN] with your own URL and bot token, respectively)
```
curl -F "url=[YOUR_WEBSITE]/api/gdurl/tgbot"'https://api.telegram.org/bot[YOUR_BOT_TOKEN]/setWebhook'
```
In this way, connect your server to your telegram bot, try to send `/help` to the bot, and if it replies to you with instructions, then the configuration is successful.

## Supplementary explanation
In the `config.js` file, there are several other parameters:
```
// How many milliseconds of a single request does not respond after a timeout (the reference value, if it continuously times out, the next time it is adjusted to twice the previous time)
const TIMEOUT_BASE = 7000

// The maximum timeout setting, such as a certain request, the first 7s timeout, the second 14s, the third 28s, the fourth 56s, the fifth time is not 112s but 60s, the same is true for subsequent
const TIMEOUT_MAX = 60000

const LOG_DELAY = 5000 // Log output interval, in milliseconds
const PAGE_SIZE = 1000 // Each network request reads the number of files in the directory. The larger the value, the more likely it will time out, and it must not exceed 1000

const RETRY_LIMIT = 7 // If a request fails, the maximum number of retries allowed
const PARALLEL_LIMIT = 20 // The number of parallel network requests, which can be adjusted according to the network environment

const DEFAULT_TARGET ='' // Required, copy the default destination ID, if you do not specify the target, it will be copied here, it is recommended to fill in the team disk ID, pay attention to use English quotation marks
```
Readers can adjust according to their respective circumstances

## Precautions
The principle of the program is to call [Google Drive Official Interface] (https://developers.google.com/drive/api/v3/reference/files/list) to recursively obtain all files and subfolder information under the target folder Roughly speaking, as many folders as there are in a certain directory, it takes at least so many requests to complete the statistics.

It is not known whether Google will limit the frequency of the interface, or whether it will affect the security of the Google account itself.

**Please don't abuse it at your own risk**
