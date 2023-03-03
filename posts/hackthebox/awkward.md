---
layout: default
title : Sayonara - Awkward Writeup
---

## Intro
In this article I’m going to be tackling `Awkward` machine, a medium difficulty Linux machine on `hackthebox.com`.

in this machine we will be preseneted with `vue js files` loaded with the page where we will search for routes and exploit a `jwt malformed token` to bypass authentication and get access to the hr dashboard, exploit `ssrf` to leak internal server resources where we will find an `api documentation` on port 3002, using this api we will exploit `awk` to get a `command injection` and extract a user's `ssh credentials` to gain initial foothold and from there exploit a `sed` command to get `command injection` and pivot to the www-data user which have access to a file that we will abuse through the `mail command` to get root privileges


![image](https://www.linkpicture.com/q/pasted_image000.png)

## Information Gathering
as always always starting by doing nmap 

![image](https://www.linkpicture.com/q/pasted_image001.png)

we see in the nmap scan output that only 2 ports are open `ssh` on port 22 and `http` on port 80 which is running `nginx server` 
let's take a look at the website running on port 80 

![image](https://www.linkpicture.com/q/pasted_image002.png)

it redirects us to `http://hat-valley.htb` so lets add this to the `/etc/hosts` file so we can access this website

![image](https://www.linkpicture.com/q/pasted_image003.png)

![image](https://www.linkpicture.com/q/pasted_image004.png)

Now you can be able to access the website

![image](https://www.linkpicture.com/q/pasted_image005.png)

using `wappalyzer` we can see that it's a nodejs application running on ubuntu's nginx server 

![image](https://www.linkpicture.com/q/pasted_image011.png)

if we navigate the website we can see that an `online store` is under building so let's fuzz the website for subdomains that may lead us to find the online store website

![image](https://www.linkpicture.com/q/pasted_image008.png)

## Subdomain FUZZING Using ffuf
`-mc all` => match all codes

![image](https://www.linkpicture.com/q/pasted_image006.png)

now let's filter by size and remove all responses with 132 size

![image](https://www.linkpicture.com/q/pasted_image007.png)

we have found the subdomain `store.hat-valley.htb`, let's add it to the `/etc/hosts` file so we can view the website

![image](https://www.linkpicture.com/q/pasted_image009.png)

the subdomain wants us to login 

![image](https://www.linkpicture.com/q/pasted_image010.png)

the application uses php since the application runs normally with the `/index.php` 

![image](https://www.linkpicture.com/q/pasted_image012.png)

## Directory Enumeration
enumerating directories on the root of the website

![image](https://www.linkpicture.com/q/pasted_image013.png)

enumerating directories on the store subdomain 

![image](https://www.linkpicture.com/q/pasted_image014.png)

seems that enemerating directories doesn't bring somthing of value so let's try enemerating the website frontend files
if we inspect the page we will find a bunch of files let's dig and find out if we are going to find something of interest 

![image](https://www.linkpicture.com/q/pasted_image015.png)

this is a `vue application` (vue is javascript frontend framework)

![image](https://www.linkpicture.com/q/pasted_image016.png)

![image](https://www.linkpicture.com/q/pasted_image017.png)

![image](https://www.linkpicture.com/q/pasted_image019.png)

let's test out those routes, we can notice that the `/leave` and `/dashboard` routes are redirecting us to `/hr` route so login in is mendatory to acces the other routes

![image](https://www.linkpicture.com/q/pasted_image018.png)

let's look for other vue files maybe we can find other routes
if we look at the directory called services we have 4 js files 
let's dig into these files and extract any useful informations so in all those files we get those routes :Cancel changes

`/api/all-leave`      returns jwt malformed

`/api/submit-leave`

`/api/login`

`/api/staff-details`  returns jwt malformed

`/api/store-status`

![image](https://www.linkpicture.com/q/pasted_image020.png)

![image](https://www.linkpicture.com/q/pasted_image021.png)

## Authentication Bypass by removing the Cookie
since It throws the error of `JWT token Malformed` I think it is a token error so let's pass this without token
intercept the request using burp and we can see a token with guest value let's remove and send the request we get invalid user 

![image](https://www.linkpicture.com/q/pasted_image022.png)

doing the same but with the `/api/staff-details` and we get a list of users data

![image](https://www.linkpicture.com/q/pasted_image023.png)

exfiltrating data from the `/api/staff-details` route in a nice format using curl and jq

![image](https://www.linkpicture.com/q/pasted_image024.png)

so we have passwords that are hashed using some algorithm we're going to use this website to know the type of the hashes `https://hashes.com/en/tools/hash_identifier` and looking at the result we can see that the hash is `SHA256`

![image](https://www.linkpicture.com/q/pasted_image025.png)

or use the `hash-identifier` command on linux

![image](https://www.linkpicture.com/q/pasted_image026.png)

now let's crack the hashes first thing to do before using `hashcat` or `john` the ripper let's use `crackstation`

![image](https://www.linkpicture.com/q/pasted_image027.png)

now let's login using those credentials in the `/hr` route that we have found previously

![image](https://www.linkpicture.com/q/pasted_image028.png)

![image](https://www.linkpicture.com/q/pasted_image029.png)

![image](https://www.linkpicture.com/q/pasted_image030.png)

let's intercept this using `burp suite` 

![image](https://www.linkpicture.com/q/pasted_image031.png)

`%22http:%2F%2Fstore.hat-valley.htb%22`  => url decode : `"http://store.hat-valley.htb"` 
![image](https://www.linkpicture.com/q/pasted_image032.png)

![image](https://www.linkpicture.com/q/pasted_image033.png)

## Server Side Request Forgery SSRF leak internal resources

![image](https://www.linkpicture.com/q/pasted_image034.png)

#### using ffuf to FUZZ all possible ports
we are escaping the " because ffuf removes them if we didn't escape them 

![image](https://www.linkpicture.com/q/pasted_image036.png)

`-fs 0` => filtering by size and removing empty responses

![image](https://www.linkpicture.com/q/pasted_image035.png)

let's take a look at the `internal port 3002`

![image](https://www.linkpicture.com/q/pasted_image037.png)

let's open it in the browser and take a deeper look 
and this the `internal api documentation` of all the routes we found previously 

![image](https://www.linkpicture.com/q/pasted_image039.png)

`/api/login` route is not vulnerable 

![image](https://www.linkpicture.com/q/pasted_image040.png)

## Abusing Awk to get local file inclusion

![image](https://www.linkpicture.com/q/pasted_image044.png)

![image](https://www.linkpicture.com/q/pasted_image045.png)

i will show u how can this syntax `awk '\{user}\' /var/www/private/leave_request.csv` get u `File Disclosure Vunerability` 

![image](https://www.linkpicture.com/q/pasted_image048.png)

![image](https://www.linkpicture.com/q/pasted_image049.png)

so if we find a way to `poison the user value` (token username) we could get a `File Disclosure Vulnerabilty`

## Poisoning The User Token
copy the user token value from the `storage -> Cookies -> http://hat-valley.htb` so if we find a way to poison the user value (token username) we could get a file disclosure vulnerabilty

![image](https://www.linkpicture.com/q/pasted_image050.png)

and then navigate to the `jwt.io` website to decode the token 

![image](https://www.linkpicture.com/q/pasted_image052.png)

in order to `poison the token` we need to make sure that the `signature is valid` and to do that we need to `crack the JWT token`
## Cracking the JWT
cracking the JWT using `hashcat` 

![image](https://www.linkpicture.com/q/pasted_image053.png)

![image](https://www.linkpicture.com/q/pasted_image054.png)

and now our token have a verified signature

![image](https://www.linkpicture.com/q/pasted_image055.png)

## Python Exploit to poison the token and get local file disclosure vulnerability

![image](https://www.linkpicture.com/q/pasted_image057.png)

![image](https://www.linkpicture.com/q/pasted_image059.png)

![image](https://www.linkpicture.com/q/pasted_image058.png)

so we can notice that `PWD=/var/www/hat-valley.htb` and since the application is a nodejs application there must be a `package.json` file so let's leak this file

![image](https://www.linkpicture.com/q/pasted_image060.png)

now we know the path to the `server.js` file let's leak this file 

![image](https://www.linkpicture.com/q/pasted_image061.png)

discolsing `nginx configuration file` for the store subdomain `(nginx configuration files always stored in /etc/nginx/sites-available/*.conf)`
what this does is if the  store subdomain is directed to `/cart` or `/product-details` return 403 (forbidden) end with `.php` require a password which exist in `/etc/nginx/conf.d/.htpasswd`

![image](https://www.linkpicture.com/q/pasted_image063.png)

![image](https://www.linkpicture.com/q/pasted_image064.png)

let's identify the hash type using `hash-identifier`

![image](https://www.linkpicture.com/q/pasted_image065.png)

now let's crack it using hashcat, but it couldn't crack it 

![image](https://www.linkpicture.com/q/pasted_image066.png)

let's move on and look for the users that exectutes `/bin/bash`

![image](https://www.linkpicture.com/q/pasted_image067.png)

let's take a look at the `.bashrc file` of the user bean (i couldn't discolse the same file for other users)
analyzing the file we can notice an alias that backups the user bean's backup and this seems interesting 

![image](https://www.linkpicture.com/q/pasted_image068.png)

![image](https://www.linkpicture.com/q/pasted_image069.png)

let's take a look at the backup_home bash script

![image](https://www.linkpicture.com/q/pasted_image070.png)

let's extract the `bean_backup_final.tar.gz` from the victim machine, and to do that we need to modify the python script and read the content of the compressed backup as bytes and save it to a file so we can decompress it without errors

![image](https://www.linkpicture.com/q/pasted_image071.png)

now let's run the python script and extract the compressed GZIP data

![image](https://www.linkpicture.com/q/pasted_image072.png)

extracting the gzip using tar, and we have the user's bean backup home 

![image](https://www.linkpicture.com/q/pasted_image073.png)

let's see if there is any hidden directories

![image](https://www.linkpicture.com/q/pasted_image074.png)

Discovering bean's credentials in his xpad directory, change directory to `.config/xpad` and read the `content-DS1ZS1` file which contains some `bean's credentials`

![image](https://www.linkpicture.com/q/pasted_image075.png)

let's try to ssh using those credentials, and we are in cool  

![image](https://www.linkpicture.com/q/pasted_image076.png)

## First Flag : 
we have found our first flag

![image](https://www.linkpicture.com/q/pasted_image077.png)

let's try if the password we have found in the .config/xpad `014mrbeanrules!#P` may be the password for the store
the username is `admin` and the password may be `014mrbeanrules!#P`

![image](https://www.linkpicture.com/q/pasted_image064.png)

![image](https://www.linkpicture.com/q/pasted_image078.png)

and we are in

![image](https://www.linkpicture.com/q/pasted_image079.png)

now let's move to the directory where the store file are existed

![image](https://www.linkpicture.com/q/pasted_image080.png)

after reading the code in the files i found 3 dangerous system calls in the cart_actions.php file where we may get command execution but if we look at the code we will find that there is blacklist on the special characters so it's not possible to get a command execution from the first 2 system calls but the last one using sed we may exploit it

![image](https://www.linkpicture.com/q/pasted_image081.png)

let's understand how sed can be exploited to inject commands
here we have replaced the first line with the command id 

![image](https://www.linkpicture.com/q/pasted_image082.png)

![image](https://www.linkpicture.com/q/pasted_image087.png)

so we can inject in the `{$item_id} field`

![image](https://www.linkpicture.com/q/pasted_image084.png)

![image](https://www.linkpicture.com/q/pasted_image088.png)

let's see where is the add item in the code 

![image](https://www.linkpicture.com/q/pasted_image085.png)

now let's go to the shop in the store web application (store.hat-valley.htb) and add some item to the cart and intercept this request using burp 

![image](https://www.linkpicture.com/q/pasted_image086.png)

now we will modify the item parameter and the action parameter where the user will be used to inject the commands and action will be delete_item so sed will be called
and if we execute this we sleep for 3 seconds

![image](https://www.linkpicture.com/q/pasted_image091.png)

generate a `reverse shell script` and put it in the `tmp directory`

![image](https://www.linkpicture.com/q/pasted_image098.png)

start listener 

![image](https://www.linkpicture.com/q/pasted_image099.png)

add_item first and then delete_item with item is `1/d'+-e+"1e+/tmp/reverseshell.sh"+'` to execute the reverse shell

![image](https://www.linkpicture.com/q/pasted_image093.png)

![image](https://www.linkpicture.com/q/pasted_image094.png)

#### Upgrading Simple Shells to Fully Interactive TTYs

![image](https://www.linkpicture.com/q/pasted_image096.png)
 
listing processes

![image](https://www.linkpicture.com/q/pasted_image097.png)

and this looks interesting it looks like its `monitoring the leave_requests.csv file`

![image](https://www.linkpicture.com/q/pasted_image100.png)

let's take a look at the leave_requests.csv file 

![image](https://www.linkpicture.com/q/pasted_image105.png)

let's use `PSPY` to `monitor linux processes without root privileges`, you can download it from this github repository `https://github.com/DominicBreuker/pspy` and then transfer it to the www-data session so we can use it there

![image](https://www.linkpicture.com/q/pasted_image101.png)

![image](https://www.linkpicture.com/q/pasted_image103.png)

![image](https://www.linkpicture.com/q/pasted_image104.png)

when we update the leave_request.csv file it invokes a `new process called mail` which runs with `root privileges` and based on what we get from the pspy we can see the mail command format that runs in the notify.sh script
so the schema of the mail command could be this `mail -s "Leave Request: " $name christine`

![image](https://www.linkpicture.com/q/pasted_image113.png)

to exploit the mail command go to `gtfobins` and search for mail to see how mail can be abused to execute scripts
`echo 'bean --exec="\!/tmp/reverseshell.sh"' >> leave_requests.csv`
before you execute this make sure you have `set up a listener to catch the reverse shell`

![image](https://www.linkpicture.com/q/pasted_image109.png)

we have successfully executed the reverse shell script using the mail command and we have spawned a root shell

![image](https://www.linkpicture.com/q/pasted_image110.png)

## Root Flag :
and here is the `root flag`

![image](https://www.linkpicture.com/q/pasted_image112.png)

hope you found this walkthrough easy to understand and follow 

Greeting From [Sayonara](https://github.com/ismail-arame)

<br> <br>
[Back To Home](../index.md)
<br>

