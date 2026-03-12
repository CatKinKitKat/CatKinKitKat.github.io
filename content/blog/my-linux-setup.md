+++
title = "My Linux Setup, or Why I don't use Arch btw"
date = 2026-03-12
[taxonomies]
tags = ["linux", "tools", "setup"]
+++

Yes. I used to use Arch. If you're like me, you like Arch but hate the install process. It's fun the first time, then it becomes bothersome for every new PC. I have a gist for that. But this post isn't about the install; it's about what I actually run and why.

Ever since ublue systems became viable I started to run Project Bluefin for Work and Bazzite on the Gaming machines. Those are reliable, similar and just get out of your way. Plus, they have secureboot. Which lets the grunts of the employeer auditing team very calm for some reason (autism?) and lets them enroll the machine within their corporate suites.

## The Desktop

**Arch Linux** with **GNOME**, just plain **GNOME**. I went through a tiling window manager phase (i3, sway) and came back to GNOME. Tiling is great for productivity until you need to share your screen on a Teams call and people ask why your desktop looks like a terminal from the 90s.

GNOME just works. It stays out of my way. I launch a terminal, a browser, and IntelliJ. That's 90% of my workflow.

## The Terminal

**Alacritty** as the terminal emulator. Fast, GPU-accelerated, configuration via a YAML file. No tabs, no splits; I use tmux for that.

**Zsh** with a minimal prompt. No oh-my-zsh; it's bloated and slow. I have a custom `.zshrc` with aliases for git, docker, and kubectl, and that's about it.

## The Editor Situation

**IntelliJ IDEA** for Java and Kotlin. I've tried VS Code for Java development multiple times. Each time I come back to IntelliJ within a week. The refactoring tools, the debugger, the database inspector; VS Code isn't there yet for JVM development, and I don't think it will be for a while.

**VS Code** for everything else: Markdown, Bash scripts, Terraform, YAML, quick edits. It's good at being a lightweight editor. It's not good at being an IDE, despite what the extensions try to do.

**Vim** for server-side editing, git commit messages, and situations where launching IntelliJ would be overkill. I know enough vim to be productive and dangerous.

## Dev Tools

- **Docker** and **Podman** (I switch between them depending on the project)
- **kubectl** and **k9s** for Kubernetes (k9s is the best TUI I've ever used)
- **Lazygit** because `git log --oneline --graph` gets old
- **Lazydocker** and it surely fits the pattern
- **SDKMAN** for managing JDK versions

## The Philosophy

I don't rice my desktop. I don't spend weekends customizing my dotfiles. My setup is boring on purpose. The goal is a development environment that launches fast, stays stable, and doesn't distract me from the actual work.

The most productive thing you can do with your development setup is stop tweaking it and start using it.

Well, I say that right after spending an afternoon writing a blog post about his setup. I see the irony. I choose to ignore it.
