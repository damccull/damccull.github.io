Title: Using Wyam on GitLab Pages with Continuous Integration
Published: 5/2/2018 20:10
Tags:
    - Wyam
    - Website
Lead: Learn to automatically generate and upload your Wyam site to GitLab Pages with only a git push.
---
^"../include-files/common-styles.md"

# Introduction
Continuous Integration with Wyam is a quest many have considered and promptly rejected because...it's hard. It's not natively supported on many platforms because it doesn't run on linux. At least not at the time of this article's publication. But take heart, brave adventurer, for all is not lost! Two possibilites exist to triumph in your quest: either wait for .NET Core support or follow this very tutorial.

Yes, you heard right! This short tutorial will have you continuously integrating your Wyam site into GitLab pages (and maybe other places with the right modifications (not covered here)) before you know it. Let's get started.

# Act I - Setting Out on the Quest
First thing's first: I'm not going to hold your hand on every detail. You're a brave adventurer and you can handle a little RTFM and basic setup.

Now, you need a [gitlab.com](https://gitlab.com) account. This will probably work on a self-hosted gitlab as well, but no promises. For this tutorial I recommend just signing up and testing it live.

You should also [read up on gitlab runners][gitlab-runner] so you understand what they are and what they can do.

Next you should [create a GitLab Pages repo][gitlab-pages] to store your generated content and display it to the world. You can fork an existing project (then delete everything) or just create a blank one.

Lastly, you will need a Windows computer of some kind, with 7-zip installed and on your PATH, to run all the fun on.

# Act II - Meeting the Monster
Now that you have a GitLab account and a Pages repo set up in it, let's get to work.

## Set up your GitLab Runner
Download the Windows version of the GitLab runner executable to a Windows machine that you want to use to actually run this stuff and open a command prompt to the location you decide to keep it.

You should, at this point, read the fine documentation on [Installing GitLab Runner on Windows][install-gitlab-runner], but just in case you are being extraordinarily lazy...

```
gitlab-runner.exe install
gitlab-runner.exe start
```
Above is recommended, but if you want to run NOT run it as the built in system account...

```
gitlab-runner.exe install --user <some.username> --password <that_user_password>
gitlab-runner.exe start
```

Easy. Next we need to register it as a 'specific runner' with GitLab.

1. Get a token for your runner. Go to your repo on GitLab, then Settings -> CI/CD and look under "Setup a specific Runner manually" for a registration token. You need that.
2. Type `gitlab-runner.exe register` and...
    1. Enter `https://gitlab.com` for the gitlab-ci coordinator
    2. Enter that token you just copied to your clipboard for the gitlab-ci token
    3. Enter some descriptive name to help you identify the computer this is running on later
    4. Enter some tags that this runner will match when running projects
    5. Enter `false` to the untagged jobs question
    6. Enter `true` when asked if you want to lock to the current project
    7. Enter `shell` for the executor
3. Open 'config.toml' for editing - should be in the same place as gitlab-runner.exe
    1. Change `executor` to 'shell' if it's not already set that way
    2. Change `shell` to 'powershell'


Now you have a runner for this repo. You can add this runner to other repos by registering it with the token from those repos.

## Assemble your CI Configuration File

Next you need to configure a `.gitlab-ci.yml` file that will act as the spellbook for your runner to perform its magic. Here is the file. You can copy and paste it as-is or you can read on and understand how it works.


```yml
before_script:
    - git submodule update --init --recursive

stages:
  - build
  - deploy


buildBlog:
  stage: build
  tags:
    - windows
  only:
    - master
  script:
    - mkdir -Force ..\Wyam
    #Set powershell to use TLS 1.0,1.1,1.2
    - '[Net.ServicePointManager]::SecurityProtocol = "tls12, tls11, tls"'
    #Get the version of latest Wyam
    - Invoke-WebRequest "https://raw.githubusercontent.com/Wyamio/Wyam/master/RELEASE" -outfile ..\Wyam\wyamversion.txt
    - $WYAMVERSION = get-content ..\wyam\wyamversion.txt
    - write-host $WYAMVERSION
    #Get and unzip the latest version of Wyam
    - Invoke-WebRequest "https://github.com/Wyamio/Wyam/releases/download/$WYAMVERSION/Wyam-$WYAMVERSION.zip" -OutFile ..\Wyam\Wyam.zip
    - 7z x "-o..\Wyam" "-aoa" ..\Wyam\Wyam.zip -r
    - ..\Wyam\wyam --output ..\output


pages:
    stage: deploy
    tags:
      - windows
    only:
      - master
    script:
      - 'if(test-path -path public) {remove-item -force public}'
      - mkdir -force public
      - copy-item -recurse ..\output\* public
    artifacts:
      paths:
        - public
```

The `before_script` section is where the whole thing starts. In our case, this line isn't likely needed unless you actually use submodules. This will tell the gitlab runner to recursively update all git submodules before doing anything else.

The `stages` section defines two stages to the script, and they will be executed in this order.

The `buildBlog` section is where the magic happens. This is the part that Windows is required for. The `stage` option tells the runner this section is the 'build' stage. We can supply `tags` here as well, which will allow gitlab-ci to filter out runners that don't support this build. I included 'windows' because this project has to be run on a windows box and, if you recall from above, we tagged the runner with 'windows' as well. The `only` option tells the runner only to pull the 'master' branch.

The `script` sub-section is where the Wyam site is actually generated. It's a series of PowerShell commands. It does this stuff:

1. Create a directory one level up called Wyam (to store wyam in). This shouldn't be in the current directory because the current directory is inside the cloned repo
2. Force this powershell instance to use TSL 1.0,1.1,1.2 instead of other cyphers. This lets us talk to github nicely
3. Get the version number of the latest Wyam release and save it to a file
4. Load that file's contents back into a variable
5. Print the version number to the console (for when you're watching the CI build)
6. Download the latest version of Wyam
7. Invoke 7zip to unzip the file
8. Run the Wyam tool to generate the site

<div class="note">Note: I really don't know why I didn't just save the Wyam version straight into a variable, but this works so I didn't want to change it.</div>


The `pages` section is a special section and must be named this way for GitLab to deploy your site to GitLab Pages.

The `stage`, `tags`, and `only` options are like above.

The `script` sub-section does the following:

1. Check if the 'public' folder exists and delete it and its contents if so
2. Recreate the 'public' directory (these steps ensure a clean build)
3. Copy all the Wyam output from its location to the 'public' directory

The `artifacts` sub-section defines what folders actually get uploaded back to the GitLab Pages site. Don't change the name from 'public' because the webserver looks for the 'public' folder to serve files from.

# Act III - Achieve Greatness Among the People

The next time you push your master branch to the GitLab repo, it should kick off a pipeline, pushing the updated source to your runner, which will compile it into a static site and push it back to GitLab Pages.

# Conclusion

The life of a brave adventurer isn't for everyone, but you have achieved greatness. Setting up Wyam for CI, in its current, non-Core state, is not terribly difficult, but can require a bit of research, trial, and error. I have done most of those for you. You may encounter a problem or two, but I am confident you can figure it out.

[gitlab-runner]:https://docs.gitlab.com/runner/ "GitLab Runner Documentation"
[gitlab-pages]:https://docs.gitlab.com/ee/user/project/pages/ "GitLab Pages Documentation"
[install-gitlab-runner]:https://docs.gitlab.com/runner/install/windows.html "Installing GitLab Runner on Windows"