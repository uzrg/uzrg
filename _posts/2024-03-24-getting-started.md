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

**The PowerShell profile for automated DevOps environment setup is available in the following repository: <a href="https://github.com/uzrg/psprofile" target="_blank">https://github.com/uzrg/psprofile</a>**

**This script automatically detects and configures your development environment with Git, Visual Studio Code, Ruby, and repository setup. It intelligently checks for existing installations across multiple common installation paths and provides comprehensive guidance for missing components.**

To clone and use the PowerShell profile, you can use any of the following methods:

**HTTPS Clone:**
```bash
git clone https://github.com/uzrg/psprofile.git
```

**SSH Clone (requires SSH key setup):**
```bash
git clone git@github.com:uzrg/psprofile.git
```

**GitHub CLI:**
```bash
gh repo clone uzrg/psprofile
```

**Download ZIP:**
Visit the repository page and click "Code" → "Download ZIP"

**Disclaimer: This script is designed for Windows OS and is provided "as is" without warranties or guarantees.**

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
