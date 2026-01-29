---
layout: single
title: "An automated checklist for computer setup"
date: 2026-01-29
---

A little while ago my laptop died once, and then twice, and in between each
failure I had to use a spare laptop. I ended up setting up my computer fully for
work 3 times in the space of a couple weeks. I decided to automate the boring
stuff and make a checklist program and a set of companion scripts for doing
things like pulling all of my repos and installing my most commonly used
libraries and programs through Homebrew.

I already use `stow` to [manage my dotfiles](https://brandon.invergo.net/news/2012-05-26-using-gnu-stow-to-manage-your-dotfiles.html?round=two)
so a lot of my configuration is taken care of with a few quick stows. But what I
really needed was an automated checklist that told me how close my configuration
was to the "target" laptop. It's a cheap declarative setup, like a vastly
simpler terraform or nix.

Here's an example checklist (with no color):

```
❯ ./laptop_checklist.sh

Setting up git ──────────────────────────────────────────────────────────
• Setup git config (git-config.sh)
✓ Pull this code from GitHub (git clone https://github.com/my-user/dotfiles.git)
✓ (file) /Users/user/.gitignore
✓ Copy global gitignore to home directory
× (directory) /Users/user/projects/project_1/
× (directory) /Users/user/projects/project_2/
× (directory) /Users/user/projects/project_3/
× Clone important repositories (clone-repos.sh)
✓ (environment variable) GITLAB_PAT
✓ (environment variable) GITHUB_PAT
✓ Export GitHub and GitLab tokens


Setting up terminal ─────────────────────────────────────────────────────
✓ (application) iTerm.app
✓ Install iTerm2 (https://iterm2.com/downloads.html)
• Import Iterm profile into Iterm
• Install frequently used applications (install-common-libraries.sh)
✓ Install OhMyZsh (install-omzsh.sh)
✓ Install Starship prompt (install-starship.sh)
✓ (file) /Users/user/.zshrc
✓ Copy .zshrc into home directory
✓ (file) /Users/user/.vimrc
✓ Install vim (install-vim.sh)


Install desktop applications ────────────────────────────────────────────
• Install VS Code (https://code.visualstudio.com/download)
✓ (file) /Users/user/Library/Application Support/Code/User/settings.json
✓ Run install-code.sh
✓ (application) Google Chrome.app
✓ (application) DBeaver.app
✓ (application) AWS VPN Client
✓ (application) Docker.app
✓ (application) logioptionsplus.app
✓ (application) Postman.app
✓ (application) Utilities/XQuartz.app
✓ (application) Visual Studio Code.app
• Install Google Chrome (https://www.google.com/chrome)
• Install docker and sign in to GitLab registry (https://www.docker.com/products/docker-desktop/)
• Install postman and sign in using lastpass (https://www.postman.com/downloads/)
• Install AWS VPN and set up with ixis-vpn-client-config.ovpn (https://ixisdigital.atlassian.net/l/cp/ctZeA341)
• Install DBeaver (https://dbeaver.io/download/)
• Install omnibug (https://chrome.google.com/webstore/detail/omnibug/bknpehncffejahipecakbfkomebjmokl)
• Install Logitech Options+ (https://www.logitech.com/en-us/software/logi-options-plus.html)
• Install MS Teams
• Install XQuartz (https://www.xquartz.org/)
• Install Talon Voice (https://talonvoice.com/)
• Install Visual Studio Code (https://code.visualstudio.com/download)
✓ Install common applications


Set up AWS credentials ──────────────────────────────────────────────────
✓ (directory) /Users/user/.aws
✓ Initialize .aws folder (install-aws.sh)
✓ (exec) ssocred
✓ (exec) aws
✓ Install AWS tools (install-aws.sh)
✓ (file) /Users/user/.aws_functions.zsh
✓ AWS authentication functions exist
```

The checklist tells me how close I am to the target, but it won't take any
action on its own. I have other scripts for that. But the thing about the other
scripts is that they:

- Have dependencies. There's a correct order, and I can't automate that as well
  because some steps require human action (like generating a new GitHub PAT).
- Are finicky. Shell scripting is not always a smooth experience, especially
  when you might have to jump between shells (if zsh isn't installed). Not to
mention that entropy affects setup scripts the same as it affects roads and
buildings. Links die, bits rot.
- Don't give you a high-level view of the current system state.

A real declarative system would compare the current state to the desired state
and then take steps to bring the computer into the desired state. Declarative
systems are hard to get right -- you have to handle all possible current states
and define how to get to the target state. When I'm running these scripts, I'm
just trying to remember the steps for getting from 0 to back to work. This is
just a bunch of setup scripts, and I'm fine with a little "meat in the loop."

All this is to say, I have the following setup:

- A bunch of separate setup scripts
- An idea of what I want the final system to look like

## The easy stuff
CLI programs and libraries are easy. First, you can usually get everything you
need through Homebrew or your package manager of choice. Second, it's easy to
test whether they're installed.

The shell environment is similarly easy to set up and check for. Environment
variables, dotfiles, these are all well-defined environment features that you
can check for.

## Getting trickier
GitHub PATs are essentially environment variables, but you can't
programmatically generate them. It's in the category of "easy to check, manual
to fix". The other main entrants in this category are applications like VS Code,
Chrome, XQuartz, and so on. These can be checked for in the few places that
MacOS stores applications.

## Reminders
Some things can neither be tested for nor installed automatically (without
significant effort). For these, I have this idea of a "reminder" in the
checklist that basically says "do this or else". But the script doesn't know the
status of it.

Examples are usually within applications, like signing into Chrome, importing
settings into VS Code, DBeaver, and configuring my mx ergo mouse in Logitech
Options.


## Putting it together
### Design

- A single script
- No config file, everything done in the script
- Easy to add/drop
- Easy to define sections
- Non-blocking. I want to see the whole status at once.

### Implementation
The program is composed of checkers and checklist items. The checkers are
functions that take some standard input (like the name of an evironment
variable) and check whether it exists, returning 0 (success) or 1 (fail).

Sections have a main status for the larger abstract concept ("Set up git") and
sub-statuses for each checklist item ("install git", "GH PAT", etc.). This is a
section that checks for personal access tokens.

```sh
echo "Setting up git ──────────────────────────────────────────────────────────"
git_pat_status=PASS
if ! check_env GITLAB_PAT; then
	git_pat_status=FAIL
fi
if ! check_env GITHUB_PAT; then
	git_pat_status=FAIL
fi
status $git_pat_status "Export GitHub and GitLab tokens"
```

The section passes unless any of its children fail, then it's a fail for the
whole section. Here, I forgot to add a check for the `git` binary, so let's add
it.

```sh
echo "Setting up git ──────────────────────────────────────────────────────────"
git_pat_status=PASS
if ! check_exec git; then
    git_pat_status=FAIL
fi
if ! check_env GITLAB_PAT; then
	git_pat_status=FAIL
fi
if ! check_env GITHUB_PAT; then
	git_pat_status=FAIL
fi
status $git_pat_status "Export GitHub and GitLab tokens"
```

To break it down:

- `check_exec` looks for a program using `which`.
- `check_env` looks for an environment variable that is defined and not an empty
  string. All of the `check_*` functions print their result to the console in
addition to calculating the success/failure.
- `status` prints a summary message, indicating pass or fail.

It's straightforward to write a new checker function. The checker should
evaluate the status of the thing to be checked, write a message to the console,
and then return a status to the user. Here's the definition of `check_env`:

```sh
check_env() {
	prefix="${dim}(environment variable) ${normal}"
	env_var_value=${!1}
	if [[ ! -z "$env_var_value" ]]; then
		status PASS "${prefix}$1"
		return 0
	else
		status FAIL "${prefix}$1"
		return 1
	fi
}
```

The checkers also use the `status` function to print messages to the user.

**Reminders** are not done by `check_*` functions. Instead, they send straight
to `status` with the value `TODO`:

```sh
status TODO "Import Iterm profile into Iterm"
```

And that's pretty much it. I wish I saw more general purpose frameworks for this
kind of thing, since I find it really helpful for remembering the pesky details
of setting up a new computer. If you know of any, let me know!

## The Gist

<script src="https://gist.github.com/charlie-gallagher/d1544d336bc11fb1d64d5b9a227fbf34.js"></script>

