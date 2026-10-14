---
title: "Biometric Git Signing and Secret-Free Developer Tokens with 1Password"
author: Donnie Page
pubDatetime: 2026-10-14T16:30:00.000Z
slug: biometric-git-signing-and-secret-free-tokens
featured: true
draft: false
tags:
  - homelab
  - 1password
  - git
  - security
  - devops
description: A practical guide to signing Git commits with Touch ID and eliminating plaintext GitHub personal access tokens using 1Password.
---

## Overview

While I was setting up service accounts and thinking through headless automation with GitHub Actions in my [previous post on eliminating plaintext secrets](/posts/1password-cli-environments-migration/), my mind naturally drifted to another critical part of the developer workflow. I started thinking about Git commit signing and how we handle developer access tokens.

If you have spent any time writing software or maintaining infrastructure, chances are your workstation has a few security skeletons in its closet. 

Maybe you have an unencrypted SSH private key sitting in your `~/.ssh` directory with a blank passphrase because entering passwords on every push gets tedious. Or maybe you have a long-lived GitHub personal access token exported directly into your shell profile, quietly waiting to be exposed in a screen share or leaked into your terminal history. 

For years, setting up cryptographically verified Git commits meant wrestling with GPG keys, esoteric terminal commands, and expired trust rings that broke your workflow at the worst possible moment. Most developers simply skipped it.

Securing your developer identity should feel natural rather than punitive. 1Password transforms this entire workflow by anchoring your identity to biometric hardware and keeping credentials in encrypted memory across two key areas.

> [!NOTE] Biometric Commit Signing
> 1Password functions as a native SSH agent. Instead of storing unencrypted private keys on your SSD, your private SSH signing key lives inside your encrypted vault. When you commit code, macOS requests a quick Touch ID tap to sign the commit in memory, awarding you GitHub's green verified badge without touching GPG.

> [!TIP] Secret-Free CLI Tokens
> Rather than pasting personal access tokens into dotfiles or shell environments, you store them as encrypted items inside a 1Password vault. Through secret reference URIs and native CLI plugins, your tools authenticate seamlessly at runtime with zero plaintext left on disk.

Whether you are pushing code to personal repositories or collaborating across enterprise teams, this setup gives you cryptographic assurance and clean workstations with minimal effort.

### Understanding Secret References
Throughout this guide, you will encounter secret reference URIs formatted as `op://Developer/Item/Field`. In 1Password syntax, each segment points to a specific layer in your vault hierarchy.

The first segment, `Developer`, represents an example vault where you might store your developer credentials and access tokens. You can replace `Developer` with whatever vault fits your setup, such as `Personal` or `Production`.

The second segment, `Item`, identifies the specific login, secure note, or API credential entry inside that vault.

The final segment, `Field`, designates the exact field containing the sensitive secret, such as `token`, `password`, or `credential`.

---

## Module 1 - Biometric Git Commit Signing with Touch ID

Cryptographically signing your Git commits proves to GitHub and your collaborators that your code truly originated from you and was not spoofed. Traditionally, this required setting up GPG keys, which many developers found cumbersome. 

Git now natively supports SSH keys for signing, and 1Password acts as your SSH agent. Your private key stays encrypted inside 1Password, and every commit is signed in memory through a biometric Touch ID prompt.

### Step 1 - Enable the 1Password SSH Agent
1. Open the 1Password desktop application.
2. Open Settings by pressing `Cmd + ,` on macOS.
3. Select the **Developer** tab.
4. Check the box labeled **Use the SSH agent**.

### Step 2 - Generate an Ed25519 Signing Key
1. In 1Password, click **New Item** and choose **SSH Key**.
2. Give the key a clear name, such as `GitHub Commit Signing Key`.
3. Select **Generate Key** and choose **Ed25519** for modern security and speed.
4. Copy the public key string starting with `ssh-ed25519`.

### Step 3 - Add Your Public Key to GitHub
1. Log in to GitHub and navigate to your account **Settings**.
2. Select **SSH and GPG keys** in the sidebar.
3. Click **New SSH key**.
4. Give it a descriptive title like `MacBook Pro 1Password Signing`.
5. Set the **Key type** dropdown to **Signing Key**.
6. Paste your public key string and click **Add SSH key**.

### Step 4 - Configure Global Git Signing
Open your terminal and run the following commands to configure Git to use 1Password for commit signing.

```bash
git config --global gpg.format ssh
git config --global gpg.ssh.program "/Applications/1Password.app/Contents/MacOS/op-ssh-sign"
git config --global commit.gpgsign true
git config --global user.signingkey "ssh-ed25519 YOUR_PUBLIC_KEY_HERE"
```

To test the integration, make a commit in any local repository. Your Mac will display a biometric prompt asking for Touch ID authorization, and your commit will be signed instantly.

---

## Module 2 - Securing GitHub Personal Access Tokens

Developers frequently require Personal Access Tokens for operations like managing repositories with the GitHub CLI, deploying packages, or automating releases. Storing these tokens in plaintext shell files is a major security risk because terminal history, shell snapshots, and system backups preserve the plaintext token.

By storing your token inside 1Password, you can reference it dynamically without ever saving it to disk.

### Step 1 - Create a GitHub Personal Access Token
1. In GitHub, open your account **Settings** and scroll down to **Developer settings**.
2. Select **Personal access tokens** and choose **Tokens (classic)**.
3. Click **Generate new token (classic)** and label it `Developer CLI Token`.
4. Select the necessary scopes based on your workflow, such as `repo`, `workflow`, and `read:org`.
5. Generate the token and copy the generated string.

### Step 2 - Save the Token in 1Password
1. Open 1Password and navigate to your chosen vault, such as `Developer`.
2. Click **New Item** and select **API Credential** or **Password**.
3. Set the title to `GitHub Personal Access Token`.
4. Paste the copied token string into the credential field.
5. Save the item.

### Step 3 - Verify In-Memory Resolution with op read
Verify that the 1Password CLI can resolve your token into memory on demand.

```bash
op read "op://Developer/GitHub Personal Access Token/credential"
```
When prompted, authorize the request with Touch ID. The 1Password CLI streams the token directly into process memory and displays it in your terminal without writing anything to disk.

---

## Module 3 - Zero-Plaintext Runtime Injection and CLI Ergonomics

Once your token is safely stored in 1Password, you have two clean ways to authenticate your developer workflows without exposing plaintext credentials.

### Approach 1 - Runtime Injection with op run
The `op run` command intercepts secret references in environment variables and replaces them with actual values inside the memory of the target command.

For example, you can list your private repositories using the GitHub CLI like this.

```bash
GH_TOKEN="op://Developer/GitHub Personal Access Token/credential" op run -- gh repo list
```
The token exists only in process memory for the brief duration of that specific command. Nothing is saved to your shell environment, and nothing is recorded in your shell history.

You can also use this pattern inside a project-specific `.env` template that only contains references.

```ini
# .env (Safe to keep in your project folder)
GH_TOKEN=op://Developer/GitHub Personal Access Token/credential
GITHUB_USER=your-username
```
Running commands with `op run --env-file=.env -- npm run build` injects the live token directly at execution time.

### Approach 2 - The 1Password Shell Plugin for GitHub CLI
1Password offers native shell plugins for popular developer tools, including the GitHub CLI (`gh`). Rather than prefixing commands with `op run`, the plugin automatically prompts for Touch ID whenever you run `gh`.

To initialize the plugin, run the setup command in your terminal.

```bash
op plugin init gh
```
Follow the interactive prompt to select your vault and your stored GitHub token. 1Password will configure an alias in your shell configuration. Moving forward, running commands like `gh pr create` or `gh repo view` triggers biometric authentication automatically.

---

## Wrapping Up

When you step back and look at your development environment, achieving strong security should not require sacrificing developer velocity or battling fragile tools.

By delegating commit signing and token management to 1Password, you eliminate the most common security traps that catch developers. Your private SSH keys remain protected inside encrypted storage, your Git commits earn the verified green badge through an effortless Touch ID tap, and your GitHub tokens never sit in plaintext inside shell dotfiles or forgotten `.env` files. 

Taking a few minutes to configure this setup gives you peace of mind across every line of code you write. You can push commits, manage releases, and automate your workflows knowing that your developer identity is cryptographically secure, completely frictionless, and anchored right at your fingertips.
