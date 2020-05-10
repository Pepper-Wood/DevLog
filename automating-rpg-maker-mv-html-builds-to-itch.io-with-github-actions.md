https://dev.to/pepperwood/automating-rpg-maker-mv-html-builds-to-itch-io-using-github-actions-3n02

# Automating RPG Maker MV HTML Builds to Itch.io with Github Actions

_Disclaimer: This is a living document, as I'm continuing to understand how to better these development processes._

I'm new to the game development scene. Even though a good handful of my friends in college were Game Dev majors, I had not tried my hand at them myself. But nonetheless, a recent hobby of mine has been browsing the [Game Jams](https://itch.io/jams) page on itch.io.

It wasn't until the [Magical Girl Game Jam](https://itch.io/jam/magical-girl-game-jam) where I saw a community post recruiting others for a team that I finally took the leap. I appreciated the guidance the others on that team provided me to both push myself to try implementing features and give assistance when I faced a blocker.

The game engine the lead programmer decided was RPG Maker MV (RMMV for short). Working with RMMV presents some interesting capabilities:
- The RMMV engine framework/engine is written in JavaScript, and the data for the game is stored in large JSON files.
- Extending game functionality usually means adding singular JavaScript files (referred to as Plugins) to override default behaviors.
- RMMV games can be exported to Windows, Mac, Linux, or Web Browser builds. But! the files in the project are already structured for Web Builds and are reconfigured for other export types.

It's easy to version control RMMV builds, although I recommend expanding the JSON files when committing (they're saved to be 1 line long, but diffs then show entire lines being changed rather than one or two entries) and compressing them back for builds.

In this walkthrough, I'll show how I've set up Github Actions + Python scripts to streamline the RMMV development process 

## Organizing the Repo

Before explaining what each file in the first two folders is doing, here is the current structure for my RPG Maker MV project repo.

```
├── .github/
│   ├── workflows/
│   │   └── itch_build.yml
├── scripts/
│   ├── format_json.py
│   └── zip_restrictions.py
├── src/
│   ├── audio/*
│   ├── data/*
│   ├── fonts/*
│   ├── icon/*
│   ├── img/*
│   ├── js/*
│   ├── movies/*
│   ├── save/*
│   ├── Game.rpgproject
│   ├── index.html
│   └── package.json
├── README.md
└── .gitignore
```

## Setting up Prettifying and Minifying JSON files

This step is not necessary for the automation process, but I find it useful when figuring out what changes were made to commits. From a high level:
- JSON files should be prettified after each commit automatically
- JSON files should be minified right before the game build is generated.

This file is written in Python 3 just due to personal preference.

### format_json.py

```py
import json
import os
import sys

rootdir = 'src'

format_type = sys.argv[1]

for subdir, dirs, files in os.walk(rootdir):
    for file in files:
        ext = os.path.splitext(file)[-1].lower()
        if ext == '.json':
            filename = os.path.join(subdir, file)
            print(filename)
            with open(filename, 'r') as json_file:
                json_object = json.load(json_file)
            f = open(filename, 'w')
            if format_type == 'prettify':
                f.write(json.dumps(json_object, indent=2))
            elif format_type == 'minify':
                f.write(json.dumps(json_object, separators=(',', ':')))
            f.close()
```

This one file handles both of these actions, due to the closeness of their implementation. The script is run as either `python3 scripts/format_json.py prettify` or `python3 scripts/format_json.py minify`. It checks all files in the specified root directory (hard-coded to 'src/') and applise the json.dumps() prettifying or minifying before resaving the files.

## zip_restrictions.py

```py
import os

rootdir = 'src'

checks_text = [
    "The ZIP file should not contain more than 500 individual files after extraction.",
    "The maximum length of a file name including path should not be greater than 240 characters long.",
    "The size of all the extracted content should not be greater than 1GB.",
    "The size any single extracted file should not be greater than 100MB."
]
checks = [True, True, True, True]

total_file_size = 0
total_count = 0
ONE_HUNDRED_MEGABYTES = 104857600
ONE_GIGABYTE = 1073741824

for subdir, dirs, files in os.walk(rootdir):
    if subdir == 'src/save':
        continue
    for file in files:
        if file == 'Game.rpgproject':
            continue
        if len(file) > 240:
            print(f"[!] Filename for {file} is {len(file-240)} characters too long!")
            checks[1] = False
        total_count = total_count + 1
        file_size = os.path.getsize(os.path.join(subdir, file))
        total_file_size += file_size
        if file_size >= ONE_HUNDRED_MEGABYTES:
            print(f"[!] {file} is {file_size} bytes!")
            checks[3] = False

if total_count > 500:
    checks[0] = False
    print(f"[!] {total_count} files total!")
if total_file_size > ONE_GIGABYTE:
    checks[2] = False
    print(f"[!] Total file size is {total_file_size} bytes!")

print("-"*20)
for i in range(len(checks)):
    if checks[i]:
        print(f"✅ {checks_text[i]}")
    else:
        print(f"❌ {checks_text[i]}")

```

Based on the HTML build restrictions documented by itch.io, this helper code will detect whether the resulting files will successfully create a build early (vs. having to see the error on your test page). The script is run as `python3 scripts/zip_restrictions.py`.

## Pre-Commit Git Hooks

// TODO 

**Update:** This section is still a work-in-progress as I figure out how to incorporate these changes into the repo. `python3 scripts/format_json.py prettify` was initially run as a separate Github Actions workflow, but that required pushing and immediately pulling changes on the branch.

Instead, `format_json.py prettify` and `zip_restrictions.py` should both be run as steps in the pre-commit git hook.

## itch_build.yml Github Action workflow

Now with the steps to make the developer experience with these data structures better, onto the actual build automation file.

```yml
name: itch.io HTML Build

on:
  push:
    branches:
      - master

jobs:
  itch_build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Set up Python
      uses: actions/setup-python@v1
      with:
        python-version: '3.x'
    - name: format-json.py
      run: python3 scripts/format_json.py minify
    - name: Zip Folder
      run: zip -r build.zip src/ -x "Game.rpgproject" "save/*"
    - uses: josephbmanley/butler-publish-itchio-action@master
      env:
        BUTLER_CREDENTIALS: ${{ secrets.BUTLER_CREDENTIALS }}
        CHANNEL: HTML
        ITCH_GAME: itch-game
        ITCH_USER: itch-user
        PACKAGE: build.zip
```

This Github Action is set to run on every push to the master branch of the repo. To reiterate what the first three steps are doing:
1. Opens the repos files to read the python scripts
2. Downloads the latest stable version of Python 3
3. Runs the prior format_json.py file set to minify the json files

Now it gets interesting. The workflow zips all the files in src/ except for Game.rpgproject and the save/ directory. As mentioned earlier, the RMMV project files are stored nearly identically to what the resulting HTML build file will be. The only difference is the HTML build will exclude the save/ folder and the Game.rpgproject file. "Deploying" a project in RPG Maker MV will just generate this file, and we would need to zip it to ship it to Itch.io. Hence, this takes care of two birds with one stone.

The fifth and final step is to use the available [butler-publish-itchio Github Action](https://github.com/josephbmanley/butler-publish-itchio-action). [Butler](https://itch.io/docs/butler/) is the commandline tool to interface with itch.io and requires a bit of extra set up in order to use with your project. You'll need to replace ITCH_GAME and ITCH_USER with the strings that apply to your projects, with the next step to get your Butler credentials.

## Setting Up Butler

You will need to follow the tutorial provided at itch.io for getting the correct Butler credentials following the "The automation-friendly way" process: https://itch.io/docs/butler/installing.html

You'll know if you have the correct credentials when the API key you generated has its source listed as `wharf`: https://itch.io/user/settings/api-keys

The [butler-publish-itchio Github Action](https://github.com/josephbmanley/butler-publish-itchio-action) page also details how to generate the proper API key and store it in your Github Secrets.

## Your First Build

By now, you have both `butler` available as a command on your computer and the credentials stored in your repo's Github Secrets. But you can't use the workflow to fire off a build just yet, if you're starting a new project from scratch. You'll need to create a blank game project in the itch.io and then use that project name to manually fire off the first build. Set the viewing status to something besides public, depending on the situation.

You can use the commands from the itch_build.yml file to do the manual deploy:
```
zip -r build.zip src/ -x "Game.rpgproject" "save/*"
butler push build.zip itch-name/itch-game:HTML
```

You should see your build appear on itch in a few moments. If you want to check the status of the deployment:
```
butler status itch-name/itch-game:HTML
```

Now that the first and only manual deployment is complete, you can commit the github workflow ymls. Anytime you push to master, a new HTML build will be fired off to replace the game the workflow is pointing to. 🎉

## Template Repository

I've started work on creating a template RPG Maker MV project both for myself and for others to use: https://github.com/Pepper-Wood/RPG-Maker-MV-Starter-Template

## In Conclusion

There's still a number of quality-of-life / developer experience improvements I have in mind for my RPG Maker MV projects. I can imagine a next step of improvement for this tutorial would be to configure this with 2 different itch.io push workflows:

- On pushes to master: Deploy a build to a restricted itch.io project to act as a "development" build for others on the team to troubleshoot. A password can be generated to distribute to beta testers as well.
- On new repo tag creation: Deploy a versioned build to a public itch.io project for the general public.

Additionally, itch.io has restrictions on the number of files an HTML build can be in order for it to run properly. RPG Maker MV does not offer different template options when starting projects. A new game project will contain hundreds of placeholder and default assets that need to be manually deleted. An extra validation step on the itch_build.yml process to count the total number of files would catch these issues earlier vs. seeing the message on the restricted project page.

All in all, I hope this guide was helpful. I'm looking forward to getting better with RMMV and publishing these processes.