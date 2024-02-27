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

Now this guide might seem a bit long, but luckily the set up should be relatively short. `rclone` supplies an easy-to-use configuration wizard we will be taking advantage of.

## Installation

The full installation guide can be found [here](https://rclone.org/install/), but luckily `rclone` supplies an easy to use script that can run on MacOS and most Linux distros:

> $ sudo -v ; curl https://rclone.org/install.sh | sudo bash

This script worked well for me on my Ubuntu Server and Linux Mint distros, but had some trouble with my Proxmox (Debian based) distro. If you run into any trouble, follow their guide on building the [Precompiled Binary](https://rclone.org/install/#linux-precompiled)

## Initial Config

As I stated previously, I'll be setting up with Google Drive. However, `rclone` works with tons of cloud providers. Check out the [supported providers](https://rclone.org/#providers) page to see who they support, as well as any specific configs for your provider of choice.

Luckily, most of the config is pretty standard between providers, so this guide should be applicable for most use cases.

### Running on a headless server?

This config guide will need access to a web browser (we'll get to this in just a second). If, like me, you are setting up `rclone` for a headless/remote server that doesn't have any sort of web browser, you have a few options under the [remote setup](https://rclone.org/remote_setup/) docs. 

I have personally found the easiest to be the [SSH tunnel](https://rclone.org/remote_setup/#configuring-using-ssh-tunnel). If you have an SSH connection available to the remote server, run the following to redirect the remote port to your local machine:

> $ ssh -L localhost:53682:localhost:53682 username@remote_server

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
Select `y` to save the configuration, and then `q` to quite from the config wizard. If everything went correctly, you should have a shiny new `rclone` directory you can interact with! Let's run a few test commands to make sure everything is working correctly.

### Testing your config

#### Making directories and adding files

`rclone` has a **ton** of different functionality. But since this is a *simple* guide after all, I'll be keeping it *simple*. Lets start off with accessing the cloud storage directory:

```
$ rclone lsf gdrive:/
```

This will list the directory contents of your cloud storage in a formatted way (you can use `ls` to view all contents). Its important to remember the format when referencing your cloud directories, i.e `gdrive:/` .

If you're cloud storage is empty, or if you've only allowed `rclone` to access data it creates, you likely won't see anything from this command. So lets add some stuff!

```
$ rclone mkdir gdrive:/test/
```

This will create a directory in your cloud storage named `test`. If you are using a cloud provider that lets you view the file structure, take a look and verify that you can see the newly created `test` folder.

```
$ rclone touch gdrive:test/test.txt
```

This will create a new empty file named `test.txt`. You should now be able to run the following and see your directory contents:

```
$ rclone ls gdrive:/
test/test.txt
```

#### Uploading and downloading files

Now lets check to make sure the upload/download process is working correctly. Make a test file to upload:

```
$ echo "This is a test file" > rclone_test.txt
```

Now try sending it to your cloud storage's `test` folder:

```
$ rclone copyto rclone_test.txt gdrive:/test/rclone_test.txt
```

You should be able to see it in the directory

```
$ rclone ls gdrive:/test/
rclone_test.txt
```

And now copy it back to your machine:

```
$ rclone copyto gdrive:/test/rclone_test.txt rclone_test_downloaded.txt
```

Double check the contents are the same:

```
$ cat rclone_test_downloaded.txt
This is a test file
```

Nice! 

`rclone` has a ton of commands available [here](https://rclone.org/commands/). If you want to use this for offsite backups, you will likely want to check out the `sync` option - this allows you to sync the contents of the source/destination, deleting and adding to the destination folder as needed.

## Encrypting your cloud storage

This is one of my favorite tools in the `rclone` toolbelt! You can use `rclone` to seamlessly encrypt and decrypt your files on the fly! Which means that after the initial config, you don't have to worry about any manual encryption or decryption. Some useful features:

 - On the fly encryption and decryption
 - Optional 2 layer password security
 - Passwords are stored in an obfuscated config, so you don't have to manually provide them
 - Ability to section off folders in your cloud storage for encryption
 - Encrypt file and folder names as well as file contents

### Before we get started

The full documentation on `crypt` configs can be found [here](https://rclone.org/crypt/). Some of this info can be confusing at first, but should make a lot more sense once we've gotten everything set up.

#### How this works

The encryption configuration, called `crypt`, is essentially a new remote that wraps around a previously configured remote path.

What does this mean? Take the example I set up previously, `gdrive`. Instead of accessing `gdrive` directly like in the previous examples, we will create a new remote, lets call it `encrypted_gdrive` for right now, and and point it to `gdrive`. For any encrypted files, we would interact with the `encrypted_gdrive` remote. So instead of running:

```
$ rclone ls gdrive:/
```

You would run:

```
$ rclone ls encrypted_gdrive:/
```

Interacting with `encrypted_gdrive` automatically encrypts/decrypts the files, whereas interacting with `gdrive` would NOT encrypted or decrypt the contents. We'll see this in action later.

You can also point your `crypt` remote to a specific directory within the wrapped remote. In fact, its best practice as I'll talk about in the next section. 

#### Folder structure

I highly recommend creating folders within your cloud storage directory specifically for encrypted backups. This makes it much easier to section encrypted and unencrypted data from each other.

This will depend on your personal preference. But lets say for example you have three categories of files:

1. Text documents
2. Photos
3. Music

And for each category, some of the files you would like to encrypt, and some you don't care to encrypt.

You might create a structure like so:

```
| Google Drive |
               | offsite_backup |
                                | encrypted_backup |
                                |                  | files
                                |                  | photos
                                |                  | music
                                | unencrypted_backup |
                                                     | files
                                                     | photos
                                                     | music
```

In this example, you could create three different `crypt` remotes, one for each folder, to interact with: `encrypted_files`, `encrypted_photos`, and `encrypted_music`. That way you could easily determine what folder on the cloud you were uploading files to - and each `crypt` would only have access to the folder you pointed it to.

Alternatively, you could create a single `crypt` remote, `encrypted_backup`, and use it for all three categories of encrypted data.

In either case, you would use the `crypt` remote to interact with your encrypted folders, and your unencrypted remote (`gdrive` in my previous example) to interact with unencrypted files.

Take a look at your data and find out what structure works best for you!

### You can always change your configs later

This might be a ton of info right off the bat! Luckily you can always edit your configurations later if you change your mind about things like folder structure. Now lets get this set up!

### Crypt config

Let's go ahead and set up a new `crypt` remote. For simplicity sake, I'm going to use the following folder structure in my example:

```
Google Drive |
             | offsite_backup |
                              | file_backup
```

`offsite_backup` will be the home to all my offsite backup files. `file_backup` will be the folder for all my encrypted file backups.

```
$ rclone mkdir gdrive:/offsite_backup/file_backup
```

To set up the `crypt` remote, run the same command from last time: `rclone config`


```
n) New remote
s) Set configuration password
q) Quit config
n/s/q> n
```
Just like last time, we will be creating a new remote
```
name> file_backup
```
Use whatever naming convention you like, I like to keep it consistent with the name of the folder on Google Drive
```
Type of storage to configure.
Enter a string value. Press Enter for the default ("").
Choose a number from below, or type in your own value
[snip]
XX / Encrypt/Decrypt a remote
   \ "crypt"
[snip]
Storage> crypt
```
Type `crypt` here
```
** See help for crypt backend at: https://rclone.org/crypt/ **

Remote to encrypt/decrypt.
Normally should contain a ':' and a path, eg "myremote:path/to/dir",
"myremote:bucket" or maybe "myremote:" (not recommended).
Enter a string value. Press Enter for the default ("").
remote> gdrive:/offsite_backup/file_backup/
```
This is the directory within your previously configured remote you want to point this configuration to. The `crypt` remote will only have access to this directory, as well as any subdirectories within it. 

So for this example, the `file_backup` remote will only be able to access the `file_backup` directory, but not the parent `offsite_backup` directory
```
How to encrypt the filenames.
Enter a string value. Press Enter for the default ("standard").
Choose a number from below, or type in your own value.
   / Encrypt the filenames.
 1 | See the docs for the details.
   \ "standard"
 2 / Very simple filename obfuscation.
   \ "obfuscate"
   / Don't encrypt the file names.
 3 | Adds a ".bin" extension only.
   \ "off"
filename_encryption> 1
```
Optional, you can encrypt the file names if you like. This is an added layer of security, but makes it more difficult to browse your files outside of `rclone`. If you are using Google Drive for example, this can make it hard to identify files by visiting drive.google.com. You would have to use `rclone ls file_backup:/` to view the file names unencrypted
```
Option to either encrypt directory names or leave them intact.

NB If filename_encryption is "off" then this option will do nothing.
Enter a boolean value (true or false). Press Enter for the default ("true").
Choose a number from below, or type in your own value
 1 / Encrypt directory names.
   \ "true"
 2 / Don't encrypt directory names, leave them intact.
   \ "false"
directory_name_encryption> 1
```
Similar to filename encryption, you can also optionally encrypt the folder names. This adds more security, but introduces even more difficulty viewing the contents outside of `rclone`.
```
Password or pass phrase for encryption.
y) Yes type in my own password
g) Generate random password
y/g> y
```
You can either supply your own password or have it generate a password for you. I like to choose my first password but its your choice here.
```
Password or pass phrase for salt. Optional but recommended.
Should be different to the previous password.
y) Yes type in my own password
g) Generate random password
n) No leave this optional password blank (default)
y/g/n> g
```
Optional second password. I recommend this for security, I like to have this one autogenerated
```
Password strength in bits.
64 is just about memorable
128 is secure
1024 is the maximum
Bits> 128
```
Generated password bit length. I recommend 128 or higher
```
Your password is: JAsJvRcgR-_veXNfy_sGmQ
Use this password? Please note that an obscured version of this
password (and not the password itself) will be stored under your
configuration file, so keep this generated password in a safe place.
y) Yes (default)
n) No
y/n>y
```
**Make sure to copy this generated password and store it securely.** If you forget either of the passwords here, it can make it tough (though not impossible, as noted later) to retrieve your encrypted files.
```
Edit advanced config? (y/n)
y) Yes
n) No (default)
y/n> n
```
Keeping it simple
```
Remote config
--------------------
[secret]
type = crypt
remote = remote:path
password = *** ENCRYPTED ***
password2 = *** ENCRYPTED ***
--------------------
y) Yes this is OK (default)
e) Edit this remote
d) Delete this remote
y/e/d> y
```

Nice! You're done with all the heavy lifting, `rclone` will handle the encryption and decryption from here on out! Lets give this thing a try

### Testing the encryption

This new `crypt` remote functions exactly like the previous remote we set up. So running commands is nearly identical. Lets give a few of these a try.

#### Upload an encrypted file

Create a test file

```
$ echo "Test encryption" > crypt_test.txt
```

Copy it to your encrypted remote

```
$ rclone copyto crypt_test.txt file_backup:/crypt_test.txt
```

#### Verify the upload

Now you can navigate to your cloud provider site to see if the encryption worked. Or, you can use your non-crypt remote. For example:

```
$ rclone ls gdrive:/
```

If you encrypted file names, you should see an **encrypted** filename:

```
offsite_backup/file_backup/d741lm77jegovbs6e98u8vntpti6h4hbvgei6p58ecjn8s3lqa6g
```

Now try the same with your encrypted remote:

```
$ rclone ls file_backup:/
```

You should see the **unencrypted** name:

```
crypt_test.txt
```

#### Verify encryption

Looking good! Now lets try to pull down the encrypted file and make sure its really encrypted. I had difficulty getting `rclone` to recognize the encrypted filename, so we are going to use `sync` here.

Create a test directory locally

```
$ mkdir test
```

Now call `sync` to sync the remote and local directories

```
$ rclone sync gdrive:/offsite_backup/file_backup/ test/
```

Now lets see what it pulled down

```
$ cd test && ls -al
total 12
drwxrwxr-x 2 user user 4096 Feb 27 12:28 .
drwxrwxr-x 3 user user 4096 Feb 27 12:26 ..
-rw-rw-r-- 1 user user   64 Feb 27 12:18 ktddgr3eng8ejdf9g7equqbnvs
```

Lets take a look at that file

```
$ cat ktddgr3eng8ejdf9g7equqbnvs
(��WNE�s0��&`��l̮���jk�{#m�ʯ$K��vcG��(�
    %���z�|��
```
Complete jibberish. Perfect! That means our encryption is working as expected. 

#### Decrypt file

Now in contrast, lets try to copy it using our fancy encrypted remote

```
$ rclone copyto file_backup:/crypt_test.txt encrypted_test.txt
```

And now take a look

```
$ cat encrypted_test.txt
Test encryption
```

Awesome! Encryption works as expected! We were able to verify:

1. The `crypt` remote can upload files to the expected folder
2. The files uploaded by the `crypt` remote are encrypted
3. Downloading the files **without** the `crypt` remote **doesn't** decrypt them
4. Downloading the files **with** the `crypt` remote **does** decrypt them

Your setup is complete, you can use this new `crypt` remote to store encrypted files in the cloud!

Check out the `rclone` list of commands [here](https://rclone.org/commands/) to see how you can use this to best fit your scenario. I expect the `rclone sync` will be handy!

## How do I access my files on a new machine?

Great question! Once you've gotten your configurations set up, its pretty simple to access on a new machine: follow the instructions again!

Lets say you set up Google Drive on machine A, and want to access those files on machine B. All you have to do is install `rclone` on machine B and set up a new remote the same way you did on machine A. You'll be able to view, upload, and download files the same way you did on machine A.

### What if they were encrypted?? And what if I forgot my password???

No sweat, `rclone` can handle that too.

You have two options:

1. Recreate the remotes
    - If you remember the passwords you used to set up the `crypt` remote on machine A, you can follow the guide and recreate both remotes as you did before, being sure to reuse the same passwords for the `crypt` on machine B
    - Be sure to select `y` when it asks if you want to supply your own password - and make sure to use the passwords in the same order as on machine A
2. Copy the `rclone.conf` file. More about `rclone.conf` [here](https://rclone.org/docs/#config-config-file)
    - You could also optionally copy the config files from machine A to machine B. You can find the config file at `~/.config/rclone/rclone.conf`. You should see an entry for every remote you created, including the `crypt`. Copy that entry and paste it into the config file on machine B. 
    - In this option, you don't need to remember the passwords. `rclone` can use the config file and access any encrypted files that you uploaded from machine A

## Always have a solid backup plan!

As super helpful and well written as this guide is, it only covers one piece of the 3-2-1 backup plan. Take a look at [this guide](https://www.veeam.com/blog/321-backup-rule.html) for more info on the 3-2-1 rule. Remember, the worst time to come up with a back up plan is when its too late!