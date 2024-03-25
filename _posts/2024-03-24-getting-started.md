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
