---
title: Getting Started
author: uzrg
date: 2024-03-24 20:55:00 +0800
categories: [Blogging, Tutorial, Scripting]
tags: [getting started, PowerShell]
pin: true
img_path: "/posts/20240324"
---

## Where Should We Start ?

Thank you for visiting "My little corner on the web" where I will try to document helpful tutorials and helpful resources for IT Systems Administration Tasks.
Since these modest web pages are hosted on github.io and built based on Jekyll .
After some consideration, the first post is to document how the site is built, someone else may want to do the same and/or I may need to build another one in the future. It helps speed up things if the process is well documented, could save some serious time….

## So What is Jekyll ?

[Jekyll](https://jekyllrb.com/) is a "static site generator" written in [Ruby](https://www.ruby-lang.org/en/) that takes [Markdown](https://www.markdownguide.org/getting-started/) text and transform it into responsive, static websites and blogs that can be freely hosted on GitHub Pages. Another Jekyll's advantage is the availability of free pre-built themes we can start from, the theme developers handled the heavy lifting so that we can only worry about customizations/branding and creating content. This website is based on the [Chirpy](https://chirpy.cotes.page/) theme. No need to know Ruby programming for this, my serious recommendation is to spend a sometime understanding [Markdown syntax](https://www.markdownguide.org/basic-syntax/) as after the site is built, it will be needed to create content.
Even though Ruby Programming is not necessary, we need to install Ruby and Gems on the local development machine so that we can test the customizations locally and push to github once satisfied with how the site look.

## So What are the steps?

1. Setup the local development environment
   ○ VSCode (Portable)
   ○ Git (Portable)
   ○ Ruby (Portable)
2. Clone the Chripy theme into a local git repository
3. Customize and test the site locally
   ○ Favicons
   ○ Avatar
   ○ Urls
   ○ Contacts, authors, etc..
   ○ Posts
4. Push to GitHub for build and deploy

We can do each step one by one, but to speed up the process will use a PowerShell script to combine step 1 and 2.
The following PowerShell download VSCode (Portable), PortableGit , Ruby Installer then install everything into a $devOpsPath\Tools directory .
###Note that this is not a fully automated process as the PortableGit and Ruby installers will launch a wizard requiring manual intervention to continue.
My recommended way to use the following PowerShell script is to turn it into a PowerShell Profile so that it is loaded each time a powershell console is launched. Note that it does try to download and install the various tools each time the $profile run, tools are installed the first time and subsequent runs only ensure the $env:path is updated.My prefered PowerShell Profile is the CurrentUserAllHosts located at ~\Documents\WindowsPowerShell\Profile.ps1

### PowerShell to Setup local dev environment and cloning Chirpy into a local repo

```PowerShell
<#
    AllUsersAllHosts       : C:\Windows\System32\WindowsPowerShell\v1.0\profile.ps1
    CurrentUserAllHosts    : ~\Documents\WindowsPowerShell\profile.ps1
#>

$Host.UI.RawUI.WindowTitle += " : " + $env:USERDOMAIN + "\" + $env:USERNAME + "@" + $env:COMPUTERNAME
# Define DevOps path
$devOpsPath = "C:\DevOps"

# Set the Name and Email variables for your github account
#these will be needed later to configure git
$userName = "yourgithubusername"
$userEmail = "youremailaddress@domain.tld"

$githubUser = [PSCustomObject]@{
    Name  = $userName
    Email = $userEmail
}

# Check if $devOpsPath exists and create it if it doesn't
if (!(Test-Path -Path $devOpsPath)) {
    New-Item -ItemType Directory -Force -Path $devOpsPath
}

# Define paths, URLs, outputs, extraction paths, and currentUser
$paths = @{
    "git"       = "$devOpsPath\Tools\PortableGit\bin"
    "vsCode"    = "$devOpsPath\Tools\VSCode\"
    "ruby"      = "$devOpsPath\Tools\Ruby\bin"
}

$urls = @{
    "vscode" = "https://update.code.visualstudio.com/latest/win32-x64-archive/stable"
    "git"    = "https://github.com/git-for-windows/git/releases/download/v2.33.0.windows.2/PortableGit-2.33.0.2-64-bit.7z.exe"
    "ruby"   = "https://github.com/oneclick/rubyinstaller2/releases/download/RubyInstaller-3.2.0-1/rubyinstaller-devkit-3.2.0-1-x64.exe"
}

$outputs = @{
    "vscode" = "$devOpsPath\Tools\vscode-portable.zip"
    "git"    = "$devOpsPath\Tools\PortableGit-2.33.0.2-64-bit.7z.exe"
    "ruby"   = "$devOpsPath\Tools\rubyinstaller-devkit-3.2.0-1-x64.exe"
}

$extractPaths = @{
    "vscode" = "$devOpsPath\Tools\VSCode"
    "git"    = "$devOpsPath\Tools\PortableGit"
    "ruby"   = "$devOpsPath\Tools\Ruby"
}

# Advanced Function to download and extract files
function downloadAndExtract {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$true)]
        [string]$url,
        [Parameter(Mandatory=$true)]
        [string]$output,
        [Parameter(Mandatory=$true)]
        [string]$extractPath
    )
    try {
        $response = Invoke-WebRequest -Uri $url -Method Head
        if ($response.StatusCode -eq 200) {
            Invoke-WebRequest -Uri $url -OutFile $output
            Write-Output "Download successful."

            # Create the extraction directory if it doesn't exist
            if (!(Test-Path -Path $extractPath)) {
                New-Item -ItemType Directory -Force -Path $extractPath
            }

            if ($output -like "*.zip") {
                # Extract the zip file
                Expand-Archive -Path $output -DestinationPath $extractPath -Force
                Write-Output "Extraction successful."
            } elseif($output -like "*7z.exe") {
                Set-Location $extractPath
                cd ..
                $output = (ls .\PortableGit-2.33.0.2-64-bit.7z.exe).FullName
                Start-Process $output -ArgumentList "x `"$output`" -o`"$extractPath`" -y" -NoNewWindow -Wait
            } elseif($output -like "*.exe") {
                # Run the installer
                Start-Process -FilePath $output -ArgumentList "/verysilent /dir=`"$extractPath`"" -Wait
                Write-Output "Installation successful."
            }
        } else {
            Write-Output "The URL is not responsive. Please check the URL."
        }
    } catch {
        Write-Output "An error occurred: $_"
    }
}

# Check if extraction paths exist and create them if they don't
$extractPaths.Values | ForEach-Object {
    if (!(Test-Path -Path $_)) {
        New-Item -ItemType Directory -Force -Path $_
    }
}

# Check if Git, VSCode, and Ruby paths exist
if ((Test-Path $paths.git) -and (Test-Path $paths.vsCode) -and (Test-Path $paths.ruby)) {
    # Add git.exe, Code.exe, and ruby.exe to the path
    $env:Path += ";$paths.git"
    $env:Path += ";$paths.vsCode"
    $env:Path += ";$paths.ruby"

    # Git, VSCode, and Ruby installed at the expected location then clone a remote project
    Set-Location $devOpsPath
    if (!(Test-Path -Path $githubUser.Name)) {
        New-Item -Name $githubUser.Name -Path . -ItemType Directory -Verbose
        git clone https://github.com/cotes2020/jekyll-theme-chirpy.git $githubUser.Name
         #git clone --recursive --verbose  https://github.com/cotes2020/jekyll-theme-chirpy.git uzrg
    } else {
        Write-Output "The directory $($githubUser.Name) already exists. Skipping the cloning operation."
    }

} else {
    # Download and extract VSCode, Git portable versions, and Ruby installer
    downloadAndExtract -url $urls.vscode -output $outputs.vscode -extractPath $extractPaths.vscode
    downloadAndExtract -url $urls.git -output $outputs.git -extractPath $extractPaths.git
    downloadAndExtract -url $urls.ruby -output $outputs.ruby -extractPath $extractPaths.ruby

    # Add Git, VSCode, and Ruby to path
    $env:Path += ";$paths.git"
    $env:Path += ";$paths.vsCode"
    $env:Path += ";$paths.ruby"

    # Git setup with git config
    # Git, VSCode, and Ruby downloaded and installed at the expected location then clone a remote project
    Set-Location $devOpsPath
    if (!(Test-Path -Path $githubUser.Name)) {
        New-Item -Name $githubUser.Name -Path . -ItemType Directory -Verbose
        git clone https://github.com/cotes2020/jekyll-theme-chirpy.git $githubUser.Name
        #git clone --recursive --verbose  https://github.com/cotes2020/jekyll-theme-chirpy.git uzrg

    } else {
        Write-Output "The directory $($githubUser.Name) already exists. Skipping the cloning operation."
    }

}

# Install Jekyll and Bundler
if (Test-Path $paths.ruby) {

cd $paths.ruby
    .\gem install jekyll bundler
} else {
    Write-Output "Ruby is not installed or not added to the PATH. Skipping the installation of Jekyll and Bundler."
}

# Find directories with .git or .github
$directoriesWithGit = @(
    Get-ChildItem -Path $devOpsPath -Directory | ForEach-Object {
        if ((Get-ChildItem -Path $_.FullName -Recurse -Include @('.git', '.github') -Directory -ErrorAction SilentlyContinue)) {
            if ($_.Name -ne 'Tools') {
                $_.FullName
            }
        }
    }
)

Write-Output "Found potential git<hub> repos:`n`n"
foreach ($dir in $directoriesWithGit) {
    Write-Output "$dir`n"
}
Write-Output "`nNavigate into the repo or  ''code  $devOpsPath\$($githubUser.Name)''  to open the project in vscode`n"

```

### Customize and test the site locally

If you added the above PowerShell to a PS Profile, then you should be able to open the repo with vscode

```Powershell

code C:\DevOps\<repo directory>

```

Once the repo is open, locate \_config.yml as a lot of customizations will be done there

○ Favicons
To customise Favicons with pictures of your own, please follow the [Customize the Favicon](https://chirpy.cotes.page/posts/customize-the-favicon/)

○ Avatar
To replace the avatar with your own pictures hosted on the site, place a 300\*300 pixels picture (mine is avatar.jpg) under assets\img then edit following lines from \_config.yml
avatar: "/<your github user name>/assets/img/avatar.jpg"
baseurl: "/<your github user name>"
The importance of <your github username> will be clear once we get to the Urls...

> My avatar and favicons are generated from a cell phone picture I took on Sunday July 11,2021 at 05:08 AM...
> {: .prompt-tip }

○ Urls
url: "https://<your github username>.github.io"

> When deployed to github pages, the url will be in the form {{ site.url }}/{{ site.baseurl }} thus if your github username is uzrg, then url of your site is [https://uzrg.github.io/uzrg](https://uzrg.github.io/uzrg/), this is the url that will be used to locate your images under assets/img
> {: .prompt-tip }
> In addition to avatar, urls there are other customizations that you will need to change including links to different social medias such as twitter (now x), facebook, github, linkedin, etc ...
> ○ authors,contact, share
> Navigate to \_data\origin to update the
> authors.yml
> contact.yml
> share.yml
> {: .prompt-tip }
> The yaml files are self explanatory, take a look and customize to your liking ...

○ Posts
Follow the [Writing a New Post](https://chirpy.cotes.page/posts/write-a-new-post/) to create new posts
