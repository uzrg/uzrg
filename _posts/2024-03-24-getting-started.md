---
title: Where Should I Start?
author: uzrg
date: 2024-03-24 20:55:00 +0800
categories: [Blogging, Tutorial, Scripting, Jekyll, Markdown]
tags: [getting started, PowerShell, Git, GitHub, Jekyll, Markdown]
pin: true
img_path: "/posts/20240324"
---

Welcome to "My little corner on the web" where I aim to document helpful tutorials and resources for IT Systems Administration Tasks.
These modest web pages are hosted on github.io and built based on Jekyll.
After some consideration, the first post documents how the site is built, in case others may want to do the same and/or I may need to build another one in the future. Having the process well documented could save some serious time….

## So What is Jekyll ?

<a href="https://jekyllrb.com/" target="_blank">Jekyll</a> is a "static site generator" written in <a href="https://www.ruby-lang.org/en/" target="_blank">Ruby</a> that takes <a href="https://www.markdownguide.org/getting-started/" target="_blank">Markdown</a> text and transforms it into responsive, static websites that can be hosted on <a href="https://pages.github.com/" target="_blank">GitHub Pages</a> free of charge. Another advantage of Jekyll is the availability of many pre-built themes to jump-start development. Indeed, theme developers have done the heavy lifting so that users only need to worry about customizations/branding and creating content for their sites or blogs. This website is based on the <a href="https://chirpy.cotes.page/" target="_blank">Chirpy</a> theme. There's no need to know Ruby programming for this, but I advise spending some time understanding understanding <a href="https://www.markdownguide.org/basic-syntax/" target="_blank">Markdown syntax</a> as it will be needed to create content after the site is built.
Even though Ruby Programming is not necessary, Ruby and Gems need to be installed on the local development machine to test the customizations locally before pushing to GitHub once satisfied with how the site looks.

## So What are the steps?

1. Setup the local development environment
   ○ VSCode (Portable)
   ○ Git (Portable)
   ○ Ruby (Portable)
2. Clone the Chirpy theme into a local git repository
3. Customize the site locally
   ○ Favicons
   ○ Avatar
   ○ Urls
   ○ Contacts, authors, share
   ○ Posts
4. Build and test site locally
5. Push to GitHub, GitHub Pages URL, Build and Deploy Actions

Each step can be done one by one, but to speed up the process, I will use a PowerShell script to combine steps 1 and 2.
The following PowerShell script downloads VSCode (Portable), PortableGit, and Ruby Installer, then installs everything into a $devOpsPath\Tools directory.
**_Note that this is not a fully automated process as the PortableGit and Ruby installers will launch a wizard requiring manual intervention to continue._**
I recommend using the following PowerShell script by turning it into a PowerShell Profile so that it is loaded each time a PowerShell console is launched. Note that it tries to download and install the various tools each time the $profile runs. Tools are installed the first time, and subsequent runs only ensure the $env:path is updated. My preferred PowerShell Profile is the CurrentUserAllHosts located at ~\Documents\WindowsPowerShell\Profile.ps1

### PowerShell to Setup local dev environment and cloning Chirpy into a local repo **_ this is for Windows OS and is provided "as is" no implied warranties or guarantees _**

```PowerShell
<#
    DevOps Environment Setup Script
    Version: 2.1
    Developer: uzrg
    Website: https://github.com/uzrg
    Email: <nospamemail@domain.tld>

    Features:
    - Configurable tool source (network share/web)
    - Git cheat sheet on demand
    - Automated tool installation
    - Repository cloning
    - Persistent configuration

    Profile Paths:
    - AllUsersAllHosts:   $PROFILE.AllUsersAllHosts
    - CurrentUserAllHosts: $PROFILE.CurrentUserAllHosts
#>

# Set Window Title with User and System Information
$Host.UI.RawUI.WindowTitle = "PowerShell - {0}\{1}@{2}" -f $env:USERDOMAIN, $env:USERNAME, $env:COMPUTERNAME

# DevOps Environment Configuration
$devOpsPath = "R:\DevOps"
$configFile = Join-Path $devOpsPath "devops_config.json"
$requiredTools = @('Git', 'VSCode', 'Ruby', 'Terraform')

# Create DevOps Directory Structure
if (-not (Test-Path -Path $devOpsPath)) {
    New-Item -ItemType Directory -Path $devOpsPath -Force | Out-Null
}

# Configuration Management
function Get-ToolSourcePreference {
    param(
        [Parameter(Mandatory=$true)]
        [string]$ConfigPath
    )

    if (Test-Path $ConfigPath) {
        try {
            $config = Get-Content $ConfigPath | ConvertFrom-Json
            return [bool]$config.UseNetworkShare
        }
        catch {
            Write-Warning "Invalid config file. Using default settings."
        }
    }

    $choice = $host.UI.PromptForChoice(
        'Tool Source Selection',
        'Where should tools be downloaded from?',
        @('&Network Share', '&Internet'),
        0
    )

    $config = @{
        UseNetworkShare = ($choice -eq 0)
        LastConfigured  = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    }

    $config | ConvertTo-Json | Out-File $ConfigPath -Force
    return $config.UseNetworkShare
}

# Get user preference
$useNetworkShare = Get-ToolSourcePreference -ConfigPath $configFile

# Tool Configuration
$toolConfig = @{
    Git = @{
        Path        = Join-Path $devOpsPath "Tools\PortableGit"
        Url         = "https://github.com/git-for-windows/git/releases/download/v2.43.0.windows.1/PortableGit-2.43.0-64-bit.7z.exe"
        Output      = Join-Path $devOpsPath "Tools\PortableGit.7z.exe"
        Executable  = "bin\git.exe"
    }
    VSCode = @{
        Path        = Join-Path $devOpsPath "Tools\VSCode"
        Url         = "https://update.code.visualstudio.com/latest/win32-x64-archive/stable"
        Output      = Join-Path $devOpsPath "Tools\vscode.zip"
        Executable  = "Code.exe"
    }
    Ruby = @{
        Path        = Join-Path $devOpsPath "Tools\Ruby"
        Url         = "https://github.com/oneclick/rubyinstaller2/releases/download/RubyInstaller-3.2.2-1/rubyinstaller-devkit-3.2.2-1-x64.exe"
        Output      = Join-Path $devOpsPath "Tools\rubyinstaller.exe"
        Executable  = "bin\ruby.exe"
    }
    Terraform = @{
        Path        = Join-Path $devOpsPath "Tools\Terraform"
        Url         = "https://releases.hashicorp.com/terraform/1.9.6/terraform_1.9.6_windows_amd64.zip"
        Output      = Join-Path $devOpsPath "Tools\terraform.zip"
        Executable  = "terraform.exe"
    }
}

# Git Configuration
$gitUserConfig = @{
    Name  = "yourgithubusername"
    Email = "youremailaddress@domain.tld"
}

# Tool Installation Function
function Install-Tool {
    param(
        [Parameter(Mandatory)]
        [string]$ToolName,
        [Parameter(Mandatory)]
        [hashtable]$Config
    )

    try {
        Write-Verbose "Checking $ToolName installation" -Verbose
        $exePath = Join-Path $Config.Path $Config.Executable

        if (Test-Path $exePath) {
            Write-Verbose "$ToolName already installed at $exePath" -Verbose
            return
        }

        Write-Verbose "Installing $ToolName..." -Verbose

        # Create tool directory
        if (-not (Test-Path $Config.Path)) {
            New-Item -Path $Config.Path -ItemType Directory -Force | Out-Null
        }

        # Network share logic
        if ($useNetworkShare) {
            $networkSharePath = Join-Path "\\dept-files\tools\DevOps\" (Split-Path $Config.Output -Leaf)
            if (Test-Path $networkSharePath) {
                Write-Verbose "Copying $ToolName from network share..." -Verbose
                Copy-Item -Path $networkSharePath -Destination $Config.Output -Force
            }
            else {
                Write-Warning "Network share copy failed for $ToolName - falling back to web download"
            }
        }

        # Web download fallback
        if (-not (Test-Path $Config.Output)) {
            Write-Verbose "Downloading $ToolName from web..." -Verbose
            Invoke-WebRequest -Uri $Config.Url -OutFile $Config.Output -UseBasicParsing
        }

        # Extract/Install tool
        switch -Wildcard ($Config.Output) {
            "*.zip" {
                Expand-Archive -Path $Config.Output -DestinationPath $Config.Path -Force
            }
            "*.7z.exe" {
                Start-Process -FilePath $Config.Output -ArgumentList "-o$($Config.Path) -y" -Wait -NoNewWindow
            }
            "*.exe" {
                Start-Process -FilePath $Config.Output -ArgumentList "/verysilent /dir=`"$($Config.Path)`"" -Wait
            }
        }

        # Verify installation
        if (Test-Path $exePath) {
            Write-Verbose "$ToolName installed successfully" -Verbose
            $env:Path += ";$($Config.Path)"
        }
        else {
            Write-Warning "$ToolName installation may have failed - executable not found"
        }
    }
    catch {
        Write-Warning "Failed to install $ToolName : $_"
    }
}

# Install/Update Tools
foreach ($tool in $requiredTools) {
    $config = $toolConfig[$tool]
    Install-Tool -ToolName $tool -Config $config
}

# Configure Git
if (Get-Command git -ErrorAction SilentlyContinue) {
    git config --global user.name $gitUserConfig.Name
    git config --global user.email $gitUserConfig.Email
}
else {
    Write-Warning "Git not available for configuration"
}

# Clone Repositories
if (Get-Command git -ErrorAction SilentlyContinue) {
    $repos = @(
        @{ Name = '<gitusername>'; Url = 'https://github.com/'<gitusername>'/'<gitusername>'.git' },
        @{ Name = 'jekyll-theme-chirpy'; Url = 'https://github.com/cotes2020/jekyll-theme-chirpy.git' }
    )

    foreach ($repo in $repos) {
        $repoPath = Join-Path $devOpsPath $repo.Name
        if (-not (Test-Path $repoPath)) {
            Set-Location $devOpsPath
            git clone $repo.Url
        }
        else {
            Write-Verbose "Repository $($repo.Name) already exists" -Verbose
        }
    }
}

# Install Ruby Gems
if (Get-Command ruby -ErrorAction SilentlyContinue) {
    gem install jekyll bundler --no-document --quiet
}
else {
    Write-Warning "Ruby not available for gem installation"
}

# Git Cheat Sheet Function
function Show-GitCheatSheet {
    $cheatSheet = @"
=================================================================
Git Essentials Cheat Sheet
=================================================================
Basic Workflow:
    git init                        Initialize new repository
    git clone <url>                 Clone a repository
    git add <file>                  Stage changes
    git commit -m "message"         Commit staged changes
    git push                        Push to remote repository
    git pull                        Update from remote

Branching:
    git branch                      List branches
    git branch <name>               Create new branch
    git checkout <branch>           Switch branches
    git merge <branch>              Merge branches
    git rebase <branch>             Rebase current branch

Undoing Changes:
    git reset --hard HEAD           Discard all local changes
    git revert <commit>             Create undo commit
    git clean -df                   Remove untracked files

Configuration:
    git config --global user.name "Your Name"
    git config --global user.email "your@email.com"

Useful Tips:
    git log --oneline --graph       Compact commit history
    git status -sb                  Short status format
    git diff --cached               View staged changes

Additional Resources:
- Official Git Book: https://git-scm.com/book/en/v2
- Visualizing Git: https://git-school.github.io/visualizing-git/
- Git Documentation: https://git-scm.com/doc
"@
    Write-Host $cheatSheet -ForegroundColor Cyan
}

# Final Setup Completion
Set-Location $devOpsPath
Write-Host "`nDevOps environment setup completed successfully!" -ForegroundColor Green
Write-Host "`nConfiguration Summary:"
Write-Host "- Tool Source: $(if ($useNetworkShare) { 'Network Share (\\dept-files\tools\DevOps\)' } else { 'Internet' })" -ForegroundColor Yellow
Write-Host "- Installation Directory: $devOpsPath" -ForegroundColor Yellow
Write-Host "`nAvailable Commands:"
Write-Host "- Show-GitCheatSheet : Display Git reference guide" -ForegroundColor Cyan
Write-Host "- Get-ChildItem       : Explore installed tools" -ForegroundColor Cyan
Write-Host "`nTo reconfigure tool sources, delete config file and rerun this script:"
Write-Host "Remove-Item $configFile" -ForegroundColor Yellow

```

### Customize the site locally

If you added the above PowerShell to a PS Profile, then you should be able to open the repo with VScode

```

code $devOpsPath\<repo directory>

```

Once the repo is open, locate \_config.yml as a lot of customizations will be done there

#### Favicons

To customise Favicons with pictures of your own, please follow the <a href="https://chirpy.cotes.page/posts/customize-the-favicon/" target="_blank">Customize the Favicon</a>

#### Avatar

To replace the avatar with your own pictures hosted on the site, place a 300\*300 pixels picture (mine is named avatar.jpg) under assets\img then edit following lines from \_config.yml

avatar: "/<your github user name>/assets/img/avatar.jpg"

baseurl: "/<your github user name>"

The importance of <your github username> will be clear once we get to the Urls section...

> My avatar and favicons are generated from a picture I took with my mobile phone on Sunday July 11,2021 at 05:08 AM...I Still think it was a beautiful sunrise!
> {: .prompt-tip }

#### Urls

url: "https://<your github username>.github.io"

> When deployed to github pages, the url will be in the form {{ site.url }}/{{ site.baseurl }} thus as example, if your github username is uzrg, then the url of your site is [https://uzrg.github.io/uzrg](https://uzrg.github.io/uzrg/), this is the url that will be used to locate and serve your images placed under assets/img
> {: .prompt-tip }
> In addition to avatar and urls, other customizations that need to be changed include links to different social medias platforms such as twitter (now x), facebook, github, linkedin, etc ...

#### authors,contact, share

> Navigate to \_data\origin to update the
> authors.yml
> contact.yml
> share.yml
> {: .prompt-tip }
> The yaml files are self explanatory, take a look and customize to your liking ...

#### Posts

Follow the <a href="https://chirpy.cotes.page/posts/write-a-new-post/" target="_blank">Writing a New Post</a> to create new posts.

### Build and test site locally

Now that the site is customized and some posts have been created, it’s time to run it locally to see how it looks. Assuming the PowerShell profile mentioned earlier was set up and ran as expected, all the infrastructure needed to run the site should be installed under $devOpsPath\Tools and the $env:path should be updated. If you are in VSCode, open the terminal or open a regular PowerShell terminal and navigate to $devOpsPath\<your github user> directory, then run the necessary commands.

Remember to replace <your github user> with your actual GitHub username

```
bundle install

```

Then

```
bundle exec jekyll serve

```

Executing the "bundle exec jekyll serve" command will generate and deploy the site on local machine on tcp port 4000. Thus the local site can be accessed by browsing to http://127.0.0.1:4000/<baseurl>/
the <baseurl> will be the value configured earlier in \_config.yml. If baseurl is empty in _config.yml then there is no baseurl in the local host url. For example, when i run \*\*\_bundle exec jekyll serve_\*\* the local site is http://127.0.0.1:4000/uzrg/ as my baseurl is "/uzrg". Please note that baserul always start with a / !

### Push to GitHub, GitHub Pages URL, Build and Deploy Actions

Assuming the local site looks ready for prime time, it can be pushed to github and deployed to be accessible via the github.io address.

- Did you add your github origin?

```
git remote add origin https://github.com/<your github user>/<your git hub project>.git


```

- Check status

```
git status
```

- Stage Changes

```
git add *
```

- Commit Changes

```
git commit -m "what did you change before this iteration"
```

- Before pushing to github, add linux support

```
bundle lock --add-platform x86_64-linux
```

```
git commit -m "another commit for linux support"
```

- Push to remote repository

```
git push -u origin master
```

For details on git commands refer to the <a href="https://git-scm.com/book/en/v2" target="_blank">Pro Git Book</a>

- Set Github Pages URL, Deploment Branch and Build and Deployment Actions
