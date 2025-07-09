---
title: Where Should I Start?
author: uzrg
date: 2024-03-24 20:55:00 +0800
categories: [Blogging, Tutorial, Scripting, Jekyll, Markdown]
tags: [getting started, PowerShell, Git, GitHub, Jekyll, Markdown]
pin: true
img_path: "/posts/20240324"
---

Welcome to this technical documentation site, where I maintain comprehensive tutorials and resources for IT Systems Administration tasks.
This website is hosted on GitHub Pages and built using Jekyll.
This initial post documents the complete site construction process, serving as a reference for others who wish to create similar sites and for future site development. Comprehensive documentation of this process can significantly reduce development time and complexity.

## What is Jekyll?

<a href="https://jekyllrb.com/" target="_blank">Jekyll</a> is a static site generator written in <a href="https://www.ruby-lang.org/en/" target="_blank">Ruby</a> that transforms <a href="https://www.markdownguide.org/getting-started/" target="_blank">Markdown</a> text into responsive, static websites that can be hosted on <a href="https://pages.github.com/" target="_blank">GitHub Pages</a> at no cost. A key advantage of Jekyll is the availability of numerous pre-built themes that accelerate development. Theme developers have completed the foundational work, allowing users to focus on customization, branding, and content creation for their sites or blogs. This website utilizes the <a href="https://chirpy.cotes.page/" target="_blank">Chirpy</a> theme. While Ruby programming knowledge is not required, I recommend investing time in understanding <a href="https://www.markdownguide.org/basic-syntax/" target="_blank">Markdown syntax</a>, as it is essential for content creation once the site is established.
Although Ruby programming is not necessary, Ruby and Gems must be installed on the local development machine to test customizations locally before deploying to GitHub.

## Implementation Steps

1. Configure the local development environment
   • Visual Studio Code
   • Git
   • Ruby
2. Clone the Chirpy theme into a local Git repository
3. Customize the site locally
   • Favicons
   • Avatar
   • URLs
   • Contacts, authors, sharing options
   • Posts
4. Build and test the site locally
5. Deploy to GitHub Pages with automated build and deployment actions

While each step can be completed individually, I have developed a PowerShell script that combines steps 1 and 2 to streamline the process.
The following PowerShell script automatically detects and configures your development environment with Git, Visual Studio Code, Ruby, and repository setup.
**Note: This script intelligently checks for existing installations across multiple common installation paths and provides comprehensive guidance for missing components.**
I recommend implementing this PowerShell script as a PowerShell Profile to ensure it loads automatically when a PowerShell console is launched. The recommended PowerShell Profile is CurrentUserAllHosts, located at ~\Documents\WindowsPowerShell\Profile.ps1

### PowerShell Script for Local Development Environment Setup and Chirpy Repository Cloning

**Disclaimer: This script is designed for Windows OS and is provided "as is" without warranties or guarantees.**

```PowerShell
<#
.SYNOPSIS
Configures a DevOps environment with Git, VSCode, Ruby, and repository setup.

.DESCRIPTION
Sets up DevOps environment: checks tools, configures Git, clones repos, provides SSH help.

.PARAMETER Force
Forces Git reconfiguration.

.PARAMETER SkipRepos
Skips repository cloning.

.NOTES
    Version: 3.3.0
    Developer: uzrg
    DISCLAIMER: Use at your own risk.
#>

[CmdletBinding()]
param([switch]$Force, [switch]$SkipRepos)

#region Configuration
$Config = @{
    DevOpsPath = Join-Path -Path $HOME -ChildPath 'DevOps'
    Tools = @{
        Git = @(
            "C:\Program Files\Git\bin\git.exe",
            "C:\Program Files (x86)\Git\bin\git.exe",
            "$env:LOCALAPPDATA\Programs\Git\bin\git.exe"
        )
        VSCode = @(
            "C:\Program Files\Microsoft VS Code\bin\code.cmd",
            "C:\Program Files (x86)\Microsoft VS Code\bin\code.cmd",
            "$env:LOCALAPPDATA\Programs\Microsoft VS Code\bin\code.cmd"
        )
        Ruby = @(
            "C:\Ruby32-x64\bin\ruby.exe",
            "C:\Ruby31-x64\bin\ruby.exe",
            "C:\Ruby30-x64\bin\ruby.exe",
            "C:\tools\ruby\bin\ruby.exe",
            "$env:LOCALAPPDATA\Programs\Ruby\bin\ruby.exe"
        )
    }
    Repos = @(
        @{Name='uzrg'; Url='git@github.com:uzrg/uzrg.git'},
        @{Name='jekyll-theme-chirpy'; Url='git@github.com:cotes2020/jekyll-theme-chirpy.git'}
    )
}
$InstallCache = @{}
#endregion

#region Core Functions
function Write-Status($Message, $Type='Info') {
    $Colors = @{Info='White'; Warning='Yellow'; Error='Red'; Success='Green'}
    $timestamp = Get-Date -Format 'HH:mm:ss'
    Write-Host "[$timestamp] $Message" -ForegroundColor $Colors[$Type]
}

function Test-Tool($Name) {
    if ($InstallCache[$Name]) { return $InstallCache[$Name] }

    # First check if tool is in PATH
    $pathTool = Get-Command $Name.ToLower() -ErrorAction SilentlyContinue
    if ($pathTool) {
        $result = @{
            Found = $true
            Path = $pathTool.Source
            BinPath = [System.IO.Path]::GetDirectoryName($pathTool.Source)
            Version = $null
        }

        # Get version info for Ruby
        if ($Name -eq 'Ruby') {
            try {
                $versionOutput = & $pathTool.Source --version 2>$null
                if ($versionOutput) {
                    $result.Version = $versionOutput -replace '^ruby\s+(\d+\.\d+\.\d+).*', '$1'
                }
            } catch {}
        }

        $InstallCache[$Name] = $result
        return $result
    }

    # Check predefined paths
    $foundPath = $Config.Tools[$Name] | Where-Object { Test-Path $_ } | Select-Object -First 1
    if ($foundPath) {
        $result = @{
            Found = $true
            Path = $foundPath
            BinPath = [System.IO.Path]::GetDirectoryName($foundPath)
            Version = $null
        }

        # Get version info for Ruby
        if ($Name -eq 'Ruby') {
            try {
                $versionOutput = & $foundPath --version 2>$null
                if ($versionOutput) {
                    $result.Version = $versionOutput -replace '^ruby\s+(\d+\.\d+\.\d+).*', '$1'
                }
            } catch {}
        }

        $InstallCache[$Name] = $result
        return $result
    }

    return @{Found=$false; Version=$null}
}

function Add-ToPath($Path) {
    $pathSep = [System.IO.Path]::PathSeparator
    if (-not ($env:Path -split $pathSep -contains $Path)) {
        $env:Path = "$Path$pathSep$env:Path"
        Write-Verbose "Added to PATH: $Path"
    }
}

function Get-ADInfo {
    try {
        if (-not (Get-Module ActiveDirectory -ListAvailable)) { return $null }
        Import-Module ActiveDirectory -ErrorAction Stop
        $User = Get-ADUser $env:USERNAME -Properties Mail, EmailAddress -ErrorAction Stop
        return @{
            Name = $User.Name
            Email = if ($User.Mail) { $User.Mail } else { $User.EmailAddress }
        }
    }
    catch {
        Write-Verbose "AD lookup failed: $_"
        return $null
    }
}

function Test-RubyGems($RubyPath) {
    try {
        $gemPath = Join-Path -Path ([System.IO.Path]::GetDirectoryName($RubyPath)) -ChildPath 'gem.cmd'
        if (-not (Test-Path $gemPath)) {
            $gemPath = Join-Path -Path ([System.IO.Path]::GetDirectoryName($RubyPath)) -ChildPath 'gem'
        }

        if (Test-Path $gemPath) {
            $gemVersion = & $gemPath --version 2>$null
            return @{
                Found = $true
                Path = $gemPath
                Version = $gemVersion
            }
        }
    } catch {}

    return @{Found=$false; Version=$null}
}

function Test-Jekyll($GemPath) {
    try {
        $jekyllVersion = & $GemPath list jekyll 2>$null | Select-String 'jekyll \('
        if ($jekyllVersion) {
            return @{
                Found = $true
                Version = ($jekyllVersion -replace '.*jekyll \(([^)]+)\).*', '$1')
            }
        }
    } catch {}

    return @{Found=$false; Version=$null}
}
#endregion

#region Main Functions
function Test-Environment {
    if ($ExecutionContext.SessionState.LanguageMode -eq "ConstrainedLanguage") {
        Write-Host @'
WARNING: Running in Constrained Language Mode!
Some features will be disabled. For full functionality:
1. Run PowerShell as Administrator
2. Execute: Set-ExecutionPolicy RemoteSigned
3. Rerun this script
'@ -ForegroundColor Yellow
    }

    try {
        $Host.UI.RawUI.WindowTitle = "PowerShell - $env:USERDOMAIN\$env:USERNAME@$env:COMPUTERNAME"
    } catch {}

    if (-not (Test-Path $Config.DevOpsPath)) {
        $null = New-Item -ItemType Directory -Path $Config.DevOpsPath -Force
        Write-Status "Created DevOps directory: $($Config.DevOpsPath)" Success
    }
}

function Test-Applications {
    Write-Status "Checking required applications..."

    $Git = Test-Tool 'Git'
    $VSCode = Test-Tool 'VSCode'
    $Ruby = Test-Tool 'Ruby'

    # Check for missing core applications
    $missingApps = @()
    if (-not $Git.Found) { $missingApps += 'Git for Windows' }
    if (-not $VSCode.Found) { $missingApps += 'Visual Studio Code' }

    if ($missingApps.Count -gt 0) {
        Write-Host "MISSING REQUIRED APPLICATIONS:`n$($missingApps | ForEach-Object { "- $_" })`n" -ForegroundColor Cyan
        Write-Host @"
ACTION REQUIRED:
1. Open Company Portal from Start Menu
2. Search for each application by name
3. Click 'Install' for each application
4. Wait for installations to complete
5. Rerun this script after installation
"@
        Read-Host "Press Enter to continue" | Out-Null
        throw "Missing required applications - setup cannot continue"
    }

    # Add core tools to PATH
    Add-ToPath $Git.BinPath
    Add-ToPath $VSCode.BinPath

    # Check Ruby and related tools
    $RubyGems = $null
    $Jekyll = $null

    if ($Ruby.Found) {
        Add-ToPath $Ruby.BinPath
        $RubyGems = Test-RubyGems $Ruby.Path
        if ($RubyGems.Found) {
            $Jekyll = Test-Jekyll $RubyGems.Path
        }
        Write-Status "Ruby found: $($Ruby.Version)" Success
    } else {
        Write-Status "Ruby not found - Jekyll development will be limited" Warning
        Write-Host @"
RUBY INSTALLATION RECOMMENDED:
Ruby is required for Jekyll development (jekyll-theme-chirpy repository).

INSTALLATION OPTIONS:
1. RubyInstaller: https://rubyinstaller.org/downloads/
2. Chocolatey: choco install ruby
3. Scoop: scoop install ruby

After installation, run: gem install jekyll bundler
"@ -ForegroundColor Yellow
    }

    Write-Status "Core applications verified" Success
    return @{
        Git = $Git
        VSCode = $VSCode
        Ruby = $Ruby
        RubyGems = $RubyGems
        Jekyll = $Jekyll
    }
}

function Set-GitConfig {
    $CurrentName = git config --global user.name 2>$null
    $CurrentEmail = git config --global user.email 2>$null

    if ($CurrentName -and $CurrentEmail -and -not $Force) {
        Write-Status "Git already configured ($CurrentName, $CurrentEmail)"
        return
    }

    $Name = $CurrentName
    $Email = $CurrentEmail

    if (-not $Name -or $Force) { $Name = Read-Host "Enter Git user name" }
    if (-not $Email -or $Force) { $Email = Read-Host "Enter Git email address" }

    if ((-not $Name -or -not $Email) -and (Get-CimInstance Win32_ComputerSystem).PartOfDomain) {
        Write-Status "Checking Active Directory..."
        $ADUser = Get-ADInfo
        if ($ADUser) {
            if (-not $Name) { $Name = $ADUser.Name }
            if (-not $Email) { $Email = if ($ADUser.Email) { $ADUser.Email } else { "nospam@domain.tld" } }
        }
    }

    if ($Name -and $Name -ne $CurrentName) {
        git config --global user.name $Name
        Write-Status "Set Git user.name: $Name" Success
    }
    if ($Email -and $Email -ne $CurrentEmail) {
        git config --global user.email $Email
        Write-Status "Set Git user.email: $Email" Success
    }

    git config --global init.defaultBranch main
    git config --global pull.rebase false
    git config --global core.autocrlf true
}

function Invoke-RepoCloning {
    if ($SkipRepos) { return }

    Write-Status "Cloning repositories..."
    $OriginalLocation = Get-Location

    try {
        Set-Location $Config.DevOpsPath

        foreach ($Repo in $Config.Repos) {
            $RepoPath = Join-Path -Path $Config.DevOpsPath -ChildPath $Repo.Name
            if (Test-Path $RepoPath) {
                Write-Status "Repository $($Repo.Name) already exists"
                continue
            }

            try {
                $null = git clone $Repo.Url 2>&1
                if ($LASTEXITCODE -eq 0) {
                    Write-Status "Cloned: $($Repo.Name)" Success

                    # Special handling for Jekyll repositories
                    if ($Repo.Name -eq 'jekyll-theme-chirpy') {
                        Show-JekyllSetupInstructions $RepoPath
                    }
                } else {
                    throw "Clone failed"
                }
            }
            catch {
                Write-Status "Failed to clone $($Repo.Name)" Error
                $UserEmail = git config --global user.email 2>$null
                Show-SSHHelp $UserEmail
            }
        }
    }
    finally {
        Set-Location $OriginalLocation
    }
}

function Show-JekyllSetupInstructions($RepoPath) {
    $Ruby = Test-Tool 'Ruby'
    if (-not $Ruby.Found) {
        Write-Host @"

JEKYLL REPOSITORY CLONED: $RepoPath

To set up Jekyll development:
1. Install Ruby: https://rubyinstaller.org/downloads/
2. Install Jekyll and Bundler: gem install jekyll bundler
3. Navigate to repository: cd '$RepoPath'
4. Install dependencies: bundle install
5. Serve locally: bundle exec jekyll serve

"@ -ForegroundColor Yellow
    } else {
        Write-Host @"

JEKYLL REPOSITORY CLONED: $RepoPath

Next steps for Jekyll development:
1. Navigate to repository: cd '$RepoPath'
2. Install dependencies: bundle install
3. Serve locally: bundle exec jekyll serve

"@ -ForegroundColor Cyan
    }
}

function Show-SSHHelp($Email = "nospam@domain.tld") {
    Write-Host @"
SSH SETUP INSTRUCTIONS:

1. GENERATE SSH KEY:
   ssh-keygen -t rsa -b 4096 -C "$Email"

2. START SSH AGENT (PowerShell):
   Start-Service ssh-agent

   OR manually:
   `$sshAgent = ssh-agent; `$sshAgent | Invoke-Expression

3. ADD PRIVATE KEY:
   ssh-add ~/.ssh/id_rsa

4. ADD PUBLIC KEY TO GITHUB:
   - Copy key: Get-Content ~/.ssh/id_rsa.pub | clip
   - GitHub Settings > SSH and GPG keys > New SSH key
   - Paste key and save

5. TEST CONNECTION:
   ssh -T git@github.com

Documentation: https://docs.github.com/en/authentication/connecting-to-github-with-ssh
"@ -ForegroundColor Yellow
}

function Show-GitCheatSheet {
    Write-Host @'

GIT ESSENTIALS CHEAT SHEET

BASIC WORKFLOW:
  git init                     ? Initialize new repository
  git clone <url>              ? Clone a repository
  git add <file>               ? Stage changes
  git commit -m "message"      ? Commit staged changes
  git push                     ? Push to remote repository
  git pull                     ? Update from remote

BRANCHING:
  git branch                   ? List branches
  git branch <name>            ? Create new branch
  git checkout <branch>        ? Switch branches
  git merge <branch>           ? Merge branches

UNDOING CHANGES:
  git reset --hard HEAD        ? Discard all local changes
  git revert <commit>          ? Create undo commit

SSH OPERATIONS:
  ssh-keygen -t rsa -b 4096    ? Generate SSH key
  Start-Service ssh-agent      ? Start SSH agent (PowerShell)
  ssh-add ~/.ssh/id_rsa        ? Add SSH key to agent

JEKYLL DEVELOPMENT:
  bundle install               ? Install Jekyll dependencies
  bundle exec jekyll serve     ? Start local development server
  bundle exec jekyll build     ? Build static site

RESOURCES:
  ? Git Book: https://git-scm.com/book/en/v2
  ? Visualizing Git: https://git-school.github.io/visualizing-git/
  ? Jekyll Docs: https://jekyllrb.com/docs/
  ? Git Documentation: https://git-scm.com/doc

'@ -ForegroundColor Cyan
}

function Show-Summary($Apps) {
    Set-Location $Config.DevOpsPath

    $repoList = if (Test-Path $Config.DevOpsPath) {
        (Get-ChildItem -Directory -Path $Config.DevOpsPath).Name | ForEach-Object { "- $_" }
    } else { "None" }

    $rubyStatus = if ($Apps.Ruby.Found) {
        "Ruby $($Apps.Ruby.Version)"
    } else {
        "Not installed"
    }

    $jekyllStatus = if ($Apps.Jekyll -and $Apps.Jekyll.Found) {
        "Jekyll $($Apps.Jekyll.Version)"
    } else {
        "Not installed"
    }

    Write-Host @"
DEVOPS ENVIRONMENT SETUP COMPLETE

CONFIGURATION SUMMARY:
- Git: $($Apps.Git.Found)
- VSCode: $($Apps.VSCode.Found)
- Ruby: $rubyStatus
- Jekyll: $jekyllStatus
- Directory: $($Config.DevOpsPath)

AVAILABLE COMMANDS:
- Show-GitCheatSheet  : Display Git reference guide

REPOSITORIES:
$($repoList -join "`n")
"@ -ForegroundColor Green
}
#endregion

#region Execution
try {
    Write-Status "Starting DevOps environment setup..." Success
    Test-Environment
    $Apps = Test-Applications
    Set-GitConfig
    Invoke-RepoCloning
    Show-Summary $Apps

    if (-not (Get-Alias GitCheatSheet -ErrorAction SilentlyContinue)) {
        Set-Alias -Name GitCheatSheet -Value Show-GitCheatSheet -Scope Global
    }

    Write-Status "Setup completed successfully!" Success
}
catch {
    Write-Status "Setup failed: $_" Error
    exit 1
}
#endregion
```

### Local Site Customization

If you have added the above PowerShell script to a PowerShell Profile, you can open the repository with Visual Studio Code using the following command:

```
code $devOpsPath\<repo directory>
```

Once the repository is open, locate the `_config.yml` file, as most customizations will be performed within this file.

#### Favicon Configuration

To customize favicons with your own images, please follow the <a href="https://chirpy.cotes.page/posts/customize-the-favicon/" target="_blank">Customize the Favicon</a> guide.

#### Avatar Configuration

To replace the default avatar with your own image hosted on the site, place a 300×300 pixel image (in this example, named `avatar.jpg`) under `assets/img`, then edit the following lines in `_config.yml`:

```yaml
avatar: "/<your github username>/assets/img/avatar.jpg"
baseurl: "/<your github username>"
```

The significance of your GitHub username will become clear in the URLs section.

> **Note:** My avatar and favicons are generated from a photograph I captured with my mobile device on Sunday, July 11, 2021, at 05:08 AM. I continue to appreciate that beautiful sunrise!
> {: .prompt-tip }

#### URL Configuration

```yaml
url: "https://<your github username>.github.io"
```

> **Important:** When deployed to GitHub Pages, the URL follows the format `{{ site.url }}/{{ site.baseurl }}`. For example, if your GitHub username is `uzrg`, your site URL will be [https://uzrg.github.io/uzrg](https://uzrg.github.io/uzrg/). This URL is used to locate and serve images placed under `assets/img`.
> {: .prompt-tip }
>
> In addition to avatar and URL configurations, other customizations include links to various social media platforms such as Twitter (now X), Facebook, GitHub, LinkedIn, etc.

#### Authors, Contact, and Share Configuration

> Navigate to `_data/origin` to update the following files:
> - `authors.yml`
> - `contact.yml`
> - `share.yml`
> {: .prompt-tip }
>
> These YAML files are self-explanatory. Review and customize them according to your preferences.

#### Post Creation

Follow the <a href="https://chirpy.cotes.page/posts/write-a-new-post/" target="_blank">Writing a New Post</a> guide to create new posts.

### Local Site Building and Testing

Once the site has been customized and initial posts have been created, the next step is to run it locally to verify its appearance and functionality. Assuming the PowerShell profile mentioned earlier has been configured and executed successfully, all required infrastructure for running the site should be installed and detected automatically, with necessary paths added to the environment variables.

If you are working in Visual Studio Code, open the integrated terminal, or alternatively, open a standard PowerShell terminal and navigate to the `$devOpsPath\<your github username>` directory, then execute the following commands.

**Note:** Replace `<your github username>` with your actual GitHub username.

```bash
bundle install
```

Then execute:

```bash
bundle exec jekyll serve
```

Executing the `bundle exec jekyll serve` command will generate and deploy the site on the local machine using TCP port 4000. The local site can be accessed by navigating to `http://127.0.0.1:4000/<baseurl>/` in your web browser.

The `<baseurl>` corresponds to the value configured in `_config.yml`. If the baseurl is empty in `_config.yml`, then no baseurl is required in the localhost URL. For example, when I execute `bundle exec jekyll serve`, my local site is accessible at `http://127.0.0.1:4000/uzrg/` because my baseurl is "/uzrg".

**Important:** The baseurl must always begin with a forward slash (/).

### GitHub Deployment and Pages Configuration

Once the local site meets your requirements and is ready for production, it can be deployed to GitHub and made accessible via the GitHub Pages URL.

Follow these steps for deployment:

1. **Configure GitHub remote origin:**
   ```bash
   git remote add origin https://github.com/<your github username>/<your github project>.git
   ```

2. **Check repository status:**
   ```bash
   git status
   ```

3. **Stage changes:**
   ```bash
   git add *
   ```

4. **Commit changes:**
   ```bash
   git commit -m "Initial site setup and customization"
   ```

5. **Add Linux platform support:**
   ```bash
   bundle lock --add-platform x86_64-linux
   ```

6. **Commit platform support:**
   ```bash
   git commit -m "Add Linux platform support for GitHub Actions"
   ```

7. **Push to remote repository:**
   ```bash
   git push -u origin main
   ```

For comprehensive Git command reference, consult the <a href="https://git-scm.com/book/en/v2" target="_blank">Pro Git Book</a>.

#### GitHub Pages Configuration

After pushing your repository to GitHub, configure GitHub Pages URL, deployment branch, and build/deployment actions through your repository settings.
