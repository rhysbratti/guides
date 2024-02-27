# Offsite Backups with RClone!

## An important piece of a solid backup strategy

A good backup strategy is vitally important to any personal or enterprise level file system or application. Unfortunately, it doesn't always get the attention it deserves until its too late. Often, it takes data loss (or the threat of data loss) to see the value of backups. Trust me, I speak from experience!

## The 3-2-1 backup strategy

The [3-2-1](https://www.veeam.com/blog/321-backup-rule.html) rule is a good rule of thumb for formulating a backup plan. It outlines to have:

- **3** copies of your data
- On **2** different storage mediums
- With **1** copy off-site

This guide is to help with the last bullet, the **offsite backup**. Specifically, storing backups with a cloud provider. I recently started using [Rclone](https://rclone.org/) for this, and the results have been fantastic.

## In this guide

I'll be covering three major points:

1. Installing `rclone`
2. Configuring `rclone` with a cloud provider (I'm using Google Drive, but `rclone` supports nearly every cloud provider on the market)
3. Encrypting your files on the cloud for added security (One of the coolest things about `rclone` in my opinion!)

## Installation

The full installation guide can be found [here](https://rclone.org/install/), but luckily `rclone` supplies an easy to use script that can run on MacOS and most Linux distros:

> sudo -v ; curl https://rclone.org/install.sh | sudo bash

This script worked well for me on my Ubuntu Server and Linux Mint distros, but had some trouble with my Proxmox (Debian based) distro. If you run into any trouble, follow their guide on building the [Precompiled Binary](https://rclone.org/install/#linux-precompiled)

## Initial Config

As I stated previously, I'll be setting up with Google Drive. However, `rcloud` works with tons of cloud providers. Check out the [supported providers](https://rclone.org/#providers) page to see who they support, as well as any specific configs for your provider of choice.

Luckily, most of the config is pretty standard between providers, so this guide should be applicable for most use cases.

### Running on a headless server?

This config guide will need access to a web browser (we'll get to this in just a second). If, like me, you are setting up `rclone` for a headless/remote server that doesn't have any sort of web browser, you have a few options under the [remote setup](https://rclone.org/remote_setup/) docs. 

I have personally found the easiest to be the [SSH tunnel](https://rclone.org/remote_setup/#configuring-using-ssh-tunnel). If you have an SSH connection available to the remote server, run the following to redirect the remote port to your local machine:

> ssh -L localhost:53682:localhost:53682 username@remote_server

This will open an SSH session and allow you to access the remote port 53682 on your local machine when the time comes.

---

### Rclone config

Run `rclone config`, and this will launch into an interactive configuration wizard. Most of the options here are explained in their guide, I'll follow a very simple setup with Google Drive.

```
No remotes found, make a new one?
n) New remote
r) Rename remote
c) Copy remote
s) Set configuration password
q) Quit config
n/r/c/s/q> n
```
Select `n` to make a new remote, but you can also edit configs later through this prompt as well
```
name> gdrive
```
This will be the name you will use to interact with your cloud storage file system later
```
Type of storage to configure.
Choose a number from below, or type in your own value
[snip]
XX / Google Drive
   \ "drive"
[snip]
Storage> drive
```
There will be a huge wall of text here listing out all of their cloud providers, I'll be selecting `drive` for Google Drive here.
```
Google Application Client Id - leave blank normally.
client_id>
Google Application Client Secret - leave blank normally.
client_secret>
```
For this config you can leave both of these blank
```
Scope that rclone should use when requesting access from drive.
Choose a number from below, or type in your own value
 1 / Full access all files, excluding Application Data Folder.
   \ "drive"
 2 / Read-only access to file metadata and file contents.
   \ "drive.readonly"
   / Access to files created by rclone only.
 3 | These are visible in the drive website.
   | File authorization is revoked when the user deauthorizes the app.
   \ "drive.file"
   / Allows read and write access to the Application Data folder.
 4 | This is not visible in the drive website.
   \ "drive.appfolder"
   / Allows read-only access to file metadata but
 5 | does not allow any access to read or download file content.
   \ "drive.metadata.readonly"
scope> 3
```
This is a specific Google Drive option (and one I really like), you can check out the [scopes](https://rclone.org/drive/#scopes) section for full details.

This essentially lets you choose how much of your Google Drive `rclone` can access and modify. I personally like option `3`. This limits `rclone` to only access data it directly creates. This not only helps with any privacy concerns, but also sections off your Google Drive data if you want to use it for anything other than offsite backups!
```
Service Account Credentials JSON file path - needed only if you want use SA instead of interactive login.
service_account_file>
```
I left this option blank. I believe this option allows you to bypass the upcoming interactive login section.
```
Edit advanced config?
y) Yes
n) No (default)
y/n> n
```
This is a simple config, so advanced config is a no no.
```
Remote config
Use web browser to automatically authenticate rclone with remote?
 * Say Y if the machine running rclone has a web browser you can use
 * Say N if running rclone on a (remote) machine without web browser access
If not sure try Y. If Y failed, try N.
y) Yes
n) No
y/n> y
```
This is where the "Running on a headless server?" section comes in handy for remote connections! You can safely choose `y` if you followed the instructions there, or if you are running this on a non-headless machine with a GUI.
```
If your browser doesn't open automatically go to the following link: http://127.0.0.1:53682/auth
Log in and authorize rclone for access
Waiting for code...
Got code
```
The link should open automatically in your browser. If running this on a remote server, you will have to copy and paste the link into your browser. 

On your browser, you will typically be asked to log in to the cloud provider you chose, as well as authorize `rclone` to access the scopes you granted it.

When you're done, you should see a page that says "Success!". You can safely close your browser and navigate back to your terminal.
```
Configure this as a Shared Drive (Team Drive)?
y) Yes
n) No
y/n> n
```
Google Drive specific, I left this option as `n`
```
--------------------
[remote]
client_id = 
client_secret = 
scope = drive
root_folder_id = 
service_account_file =
token = {"access_token":"XXX","token_type":"Bearer","refresh_token":"XXX","expiry":"2014-03-16T13:57:58.955387075Z"}
--------------------
y) Yes this is OK
e) Edit this remote
d) Delete this remote
y/e/d> y
```
Select `y` to save the configuration, and then `q` to quite from the config wizard. If everything went correctly, you should have a shiny new `rclone` directory you can interact with! Let's run a few test commands to make sure everything is working correctly

### Testing your config

`rclone` has a **ton** of different functionality. But since this is a *simple* guide after all, I'll be keeping it *simple*. Lets start off with accessing the cloud storage directory:

> $ rclone lsf gdrive:/

This will list the directory contents of your cloud storage in a formatted way (you can use `ls` to view all contents). Its important to remember the format when referencing your cloud directories, i.e `gdrive:/` .If you're cloud storage is empty, or if you've only allowed `rclone` to access data it creates, you likely won't see anything from this command. So lets add some stuff!

> $ rclone mkdir gdrive:/test/

This will create a directory in your cloud storage named `test`. If you are using a cloud provider that lets you view the file structure, take a look and verify that you can see the newly created `test` folder.

> $ rclone touch gdrive:test/test.txt

This will create a new empty file named `test.txt`. You should now be able to run `rclone ls gdrive:/` and see your directory contents:

> $ test/test.txt

Now lets check to make sure the upload/download process is working correctly. Make a test file to upload:

> $ echo "This is a test file" > rclone_test.txt

Now try sending it to your cloud storage's `test` folder

> $ rclone copyto rclone_test.txt gdrive:/test/rclone_test.txt

And now copy it back to your machine:

> $ rclone copyto gdrive:/test/rclone_test.txt rclone_test_downloaded.txt

Double check the contents are the same:

> $ cat rclone_test_downloaded.txt \
> This is a test file

Nice! 

Rclone has a ton of commands available [here](https://rclone.org/commands/). If you want to use this for offsite backups, you will likely want to check out the `sync` option - this allows you to sync the contents of the source/destination, deleting and adding to the destination folder as needed.

## Encrypting your cloud storage

This is one of my favorite tools in the `rclone` toolbelt! You can use `rclone` to seemlessly encrypt and decrypt your files on the fly! Which means that after the initial config, you don't have to worry about any manual encryption or decryption. Some useful features:

 - On the fly encryption and decryption
 - Optional 2 layer password security
 - Passwords are stored in an obfuscated config, so you don't have to manually provide them
 - Ability to section off folders in your cloud storage for encryption
 - Encrypt file and folder names as well as file contents

### Before we get started

#### How this works

The encryption configuration, called `crypt` is essentially a new remote that wraps around a previously configured remote. What does this mean? Take the example I set up previously, `gdrive`. Instead of accessing `gdrive` directly like in the previous examples, we will create a new remote, lets call it `encrypted_gdrive`, and interact with it instead. So instead of running:

> $ rclone ls gdrive:/

We would run:

> $ rclone ls encrypted_gdrive:/

#### Folder structure



