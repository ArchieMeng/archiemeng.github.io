---
title: "How to Build Your Own Repository with GitHub"
description: 
date: 2025-05-17T22:47:25+08:00
image: 
math: 
license: 
hidden: false
comments: false
isCJKLanguage: false
draft: false
lastmod: 2025-05-23T21:10:48+08:00
---

## Introduction  
Developers who have modified open-source software know that modified packages need to be distributed for installation. The most efficient approach is to build a dedicated package repository, which integrates with system package managers and ensures timely updates. Examples are Arch Linux’s ArchCN and Debian’s PPA (Personal Package Archive).  

Traditional repository setups often require complex tools or systems like Open Build Service (OBS). However, this guide focuses on lightweight methods suitable for distributing self-maintained packages.  

While this article specifically covers Debian `.deb` packages, the principles can be adapted to other package management systems.  

## Core Idea  
Use `reprepro` to create a software repository. After packaging your software, import it into the repository using `reprepro`. Host the repository via HTTP (e.g., GitHub Pages) to make it publicly accessible.  

## Direct Method  
GitHub Pages provides free static site hosting. You can host your repository as a Git repository on GitHub and use GitHub Pages to serve it. For setup details, refer to [SetupWithReprepro](https://wiki.debian.org/DebianRepository/SetupWithReprepro). Below are the simplified repo setup steps using `reprepro`:  

### Key Steps:  

- Generate a GPG signing key:

    ```bash
    gpg --gen-key
    ```
    
- Get the private key fingerprint (e.g., `9FCD68DE8EFE8EC94ABEFDCB4FC5CF6FC16FDEF3`):  
  ```bash
  $ gpg --list-secret-keys sign@example.com
  sec   ed25519 2025-05-17 [SC] [expires: 2028-05-16]
        9FCD68DE8EFE8EC94ABEFDCB4FC5CF6FC16FDEF3
  uid           [ultimate] Signing bot <sign@example.com>
  ssb   cv25519 2025-05-17 [E] [expires: 2028-05-16]
  ```
  
- Export the public key for users to verify signatures:  
  ```bash
  gpg --export --armor 9FCD68DE8EFE8EC94ABEFDCB4FC5CF6FC16FDEF3 > public.key
  ```

- Create repository directories:
   ```bash
   mkdir -p /path/to/your/repo
   mkdir -p /path/to/your/repo/conf
   ```

- Configure `/path/to/your/repo/conf/distributions`:
   ```
   Codename: <release-name>
   Suite: <release-pseudonym>
   Architectures: source i386 amd64 <...>
   Components: main <...>
   Contents:
   SignWith: <fingerprint>
   Origin: <Your project name>
   Label: <Your project name>
   Description: <Your project description>
   ```
   For configuration details, see [Configure reprepro](https://wiki.debian.org/DebianRepository/SetupWithReprepro#Configure_reprepro).

- Create `/path/to/your/repo/conf/options`:

    ```bash
    cd /path/to/your/repo
    
    # Calling reprepro will behave normally at first...
    reprepro ...
    
    # ... set some default options ...
    cat > conf/options <<EOF
    verbose
    ask-passphrase
    EOF
    
    # ... now it acts like you called `reprepro --verbose --ask-passphrase`:
    reprepro ...
    ```

- Build `.deb` packages.

- Add packages to the repository. Choose one from the following two methods according to your case.

    - Use `.changes` files:  
      
      ```bash
      reprepro -b /path/to/your/repo include <release-name> <package>.changes
      ```
      
    - Use standalone `.deb` files:  
      ```bash
      reprepro -b /path/to/your/repo includedeb <release-name> <package>.deb
      ```
    
    > Use `--ignore wrongdistribution` if package codename mismatches.

- Commit the repository to Git and push to GitHub.

- Enable GitHub Pages for the repository branch. Users can then add your repository with:  

```
deb [signed-by=/path/to/public.key] https://username.github.io/repo-name release-name component
```
Replace placeholders accordingly.  

## Workflow Method  
The direct method works for small packages, but GitHub’s [file size limits](https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-large-files-on-github) (100 MB/file) and [Git LFS quotas](https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-storage-and-bandwidth-usage#storage-quota) make it impractical for large packages. Instead, use GitHub Actions to build and host packages dynamically via GitHub Pages.

### Package Build Workflow Example  
**[From Droidian Sony Pdx206 Kernel Repository](https://github.com/ArchieMeng/kernel_sony_sm8250/blob/78703e21ebb86fb0b6e5bc91d8c7b671696f0bd0/.github/workflows/build.yml)**:  
```yaml
name: Build CI

on:
  push:
    branches: [ "droidian" ]
  pull_request:
    branches: [ "droidian" ]

# Sets permissions of the GITHUB_TOKEN to allow deployment to GitHub Pages
permissions:
  contents: read
  pages: write
  id-token: write

# Allow only one concurrent deployment, skipping runs queued between the run in-progress and latest queued.
# However, do NOT cancel in-progress runs as we want to allow these production deployments to complete.
concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4
    - name: Make output dir
      run: mkdir -pv /tmp/results
    - name: Build packages in the container
      run: |
        docker run \
        -v ${{ github.workspace }}:/sources -v /tmp/results:/results \
        quay.io/droidian/build-essential:current-amd64 /bin/bash -c '''
        apt-get install linux-packaging-snippets
        cd /sources
        RELENG_HOST_ARCH="arm64" releng-build-package
        mv -v ../*.deb ../*.buildinfo ../*.changes /results
        '''
    - uses: actions/upload-artifact@v4
      with:
        path: /tmp/results
    - name: Generate files for Github Page of this repo
      run: |
        cd /tmp/results
        ls -1 > files

        # Generate index.html
        # Get the current working directory
        dir=$(pwd)

        # Create an index.html file
        echo '<!DOCTYPE html><html><head><title>Index</title></head><body>' > index.html

        # Loop through all files in the directory
        for file in *; do
            # Check if the file is not the index.html file itself
            if [ $file != "index.html" ]; then
                # Add a link to the file in the index.html file
                echo "<a href=\"$file\">$file</a><br>" >> index.html
            fi
        done

        # Close the HTML tags
        echo "</body></html>" >> index.html
    - name: Upload pages artifact
      uses: actions/upload-pages-artifact@v3
      with:
        path: /tmp/results
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

This section uses the `releng` build tool provided by Droidian (which I later discovered is part of the gbp/Git Build Package toolchain) for packaging. For regular Debian packages, you can also use `dpkg-buildpackage` directly. The major difference between these two methods is that, `releng` will use git informations as version name while `dpkg-buildpacakge` will strictly follow the version name recorded in `debian/changelog`. Signing during this stage is optional since packages will be signed together during repository import stage.

```yaml
- name: Generate files for Github Page of this repo
  run: |
    cd /tmp/results
    ls -1 > files

    # Generate index.html
    # Get the current working directory
    dir=$(pwd)

    # Create an index.html file
    echo '<!DOCTYPE html><html><head><title>Index</title></head><body>' > index.html

    # Loop through all files in the directory
    for file in *; do
        # Check if the file is not the index.html file itself
        if [ $file != "index.html" ]; then
            # Add a link to the file in the index.html file
            echo "<a href=\"$file\">$file</a><br>" >> index.html
        fi
    done

    # Close the HTML tags
    echo "</body></html>" >> index.html
```

This step generates a directory listing page to display build artifacts when accessing the repository subpath directly.

### Repository Build Workflow Example  

This process is significantly simpler than manual repository setup thanks to the pre-existing [morph027/apt-repo-action@v3.6](https://github.com/morph027/apt-repo-action) workflow. You only need to:

1. **Configure essential parameters**:  
   Set repository metadata in workflow environment variables
2. **Handle cryptographic signing**:  
   Store your private GPG key in GitHub Secrets (`SIGNING_KEY`)
3. **Key distribution**:  
   Export the public key using:  
   
   ```bash
   gpg --export --armor [fingerprint] > public.key

> Replace `fingerprint` with the actual one of your public key.

**[Workflow from ArchieMeng/custom-debs](https://github.com/ArchieMeng/custom-debs/blob/2b8c360fefad80d80f553d101f84772b6b87d72b/.github/workflows/generate.yml)**:  

```yaml
name: Generate and Deploy Debian Repo

on:
  schedule:
    - cron: "0 23 * * *"  # run daily
  push:
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

env:
  CODENAME: trixie
  COMPONENTS: main
  ARCHITECTURES: "arm64"

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact_id: ${{ steps.upload-artifact.outputs.artifact-id }}
      keyring: ${{ steps.create-apt-repo.outputs.keyring }}
    steps:
      - name: Extract repository name
        id: repo-info
        run: |
          # Split 'owner/repo' string into its components
          echo "REPO_NAME=$(echo ${{ github.repository }} | cut -d'/' -f2)" >> $GITHUB_ENV

      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Pages
        uses: actions/configure-pages@v4

      - name: Create Packages
        run: |
          mkdir -p packages
          while read package_url; do
            echo "Processing package URL: $package_url"
            package_base_url="$package_url"
            files_url="$package_base_url/files"

            # Download the "files" list
            wget -q "$files_url" -O files.list
            if [ -s "files.list" ]; then # Check if files.list exists and is not empty
              while read package_file; do
                package_file_url="$package_base_url/$package_file"
                echo "Downloading package: $package_file_url"
                wget -q "$package_file_url" -P packages
                if [ ! -f "packages/$package_file" ]; then
                  echo "Error downloading package file: $package_file_url"
                  exit 1 # Exit the workflow on error
                fi
              done < files.list
              rm files.list # Clean up files.list
            else
              echo "Error: Could not download or empty files.list from $files_url"
              exit 1 # Exit the workflow on error
            fi
          done < package_url.list
      - name: Create APT Repository
        id: create-apt-repo
        uses: morph027/apt-repo-action@v3.6
        with:
          repo-name: ${{ env.REPO_NAME }}
          scan-dir: ./packages  # Use the 'scan-dir' option
          signing-key: ${{ secrets.SIGNING_KEY }}
          codename: ${{ env.CODENAME }}
          components: ${{ env.COMPONENTS }}
          architectures: ${{ env.ARCHITECTURES }}
          import-from-repo-url: "https://archiemeng.github.io/custom-debs"


      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v3
        with:
          name: github-pages
          path: ${{ steps.create-apt-repo.outputs.dir }}

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```
To properly use this workflow, you must configure these essential elements:

- Workflow Environment Variables

  ```yaml
  env:
    CODENAME: trixie
    COMPONENTS: main
    ARCHITECTURES: "arm64"
  ```

- Repository Secret
  Configure the GPG signing key via:
  `Project Settings > Security > Secrets and variables > Actions`

  - Secret name: `SIGNING_KEY`
  - Value: ASCII-armored GPG private key (export with `gpg --export-secret-keys -a <fingerprint>`)

- GitHub Pages Deployment Rules
  Update branch protection settings with `Project Settings > Environments > github-pages > Deployment branches`. Adjust deployment rules to match your workflow requirements:

  It should be like this:

  ![image-20250517213815932](assets/image-20250517213815932.png)

## Conclusion  
These are the two methods I've used for setting up simple software repositories on GitHub. If you don't need to distribute very large packages or lack a dedicated package build workflow, the first, simplest, and most direct method is suitable. However, if you're like me and need to distribute large packages and have a complete build and distribution workflow, you might consider trying the second method, which is entirely workflow-based.

Naturally, these methods aren't without their limitations. For instance, the process of managing packages within these self-hosted repositories doesn't inherently handle inter-package dependencies. But for small repositories of this size, this situation typically isn't a concern. :)