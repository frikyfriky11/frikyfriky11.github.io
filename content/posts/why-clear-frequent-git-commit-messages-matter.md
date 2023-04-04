+++
date = 2023-04-04T06:46:32+02:00
title = "Why clear and frequent Git commit messages matter"
description = "Git commits are not just a way to save your work. They are a powerful tool to document the code and the process around it."
authors = ["Stefano Previato"]
tags = ["Git", "commit", "documentation"]
categories = ["Git", "Programming"]
+++

Git is one of the most popular and powerful tools for version control and collaboration among developers. It allows you to track changes, revert to previous states, and merge code from different branches. However, to make the most of Git, you need to write **clear** and **frequent** commit messages that describe what you have done and why.

## What is a commit message?

A commit message is a text that accompanies each commit you make in Git. It has two parts: a **title** and an **optional body**. The title is a short summary of the changes you have made, usually no more than 50 characters. The body is a more detailed explanation of the changes, the motivation behind them, and any links to external references.

A typical commit message looks like this:

    [ACME-1] Add logging

    Add logging to controller so that access analysis is possible

The title starts with a ticket number (ACME-1) that links to the issue tracker, followed by a brief description of what the commit does (_Add logging_). The body provides more context and details about the change (_Add logging to controller so that access analysis is possible_).

## Why should you write clear commit messages?

Writing clear commit messages has many benefits for yourself and others who work on the same project. Some of them are:

- It makes it **easier to understand what each commit does and why it was done**. This can help you review your own code, debug issues, or find the source of a bug.
- It makes it **easier to navigate the history of the project** and see how it evolved over time. You can use commands like `git log` or `git show` to see the commit messages and their associated changes.
- It makes it **easier to collaborate with other developers and reviewers**. They can see what you have done and provide feedback or suggestions. It also helps avoid conflicts and merge issues by communicating your intentions clearly.
- It makes it **easier to document your project** and generate changelogs or release notes. You can use tools like `git log --oneline` or `git shortlog` to generate summaries of your commits.

## Why should you commit frequently and in small packages?

Another best practice for Git is to commit **frequently** and in **small packages** of files. This means that you should make a commit whenever you complete a logical unit of work, such as implementing a feature, fixing a bug, or refactoring some code. You should also avoid committing too many files or changes in one commit, as this can make it hard to review or revert.

Committing frequently and in small packages has several advantages:

- It makes your commits more **focused and coherent**. Each commit has a single purpose and a clear message that describes it. This makes it easier to understand what each commit does and why it was done.
- It makes your commits more **granular and reversible**. You can easily undo or modify a specific change without affecting other parts of the code. You can also use commands like `git bisect` or `git blame` to find the exact commit that introduced a bug or changed a line of code.
- It makes your commits more **manageable and mergeable**. You can avoid large and complex commits that are hard to review or merge. You can also resolve conflicts more easily by merging smaller chunks of code.

## How to write clear and frequent commit messages?

Here are some tips on how to write clear and frequent commit messages:

- **Start with a capital letter and do not end with punctuation**. This follows the convention of most Git projects and makes your messages more readable.
- **Use a ticket number or a prefix** if applicable. This helps link your commits to the issue tracker or the branch name. For example, you can use `[ACME-1]` or `[feature/logging]` as prefixes for your commit titles.
- **Write a short and descriptive title**. The title should summarize what the change does in 50 characters or less. It should answer the question: "_What does this commit do?_"
- **Write an optional body with more details**. The body should explain why the change was done and how it affects the project. You can use bullet points, paragraphs, or code snippets to illustrate your points. The body should answer the question: "_Why is this change necessary?_"
- **Use a blank line to separate the title and the body**. This helps Git and other tools to parse the commit message correctly.
- **Use a consistent style and format for your commit messages**. You can follow a convention like Conventional Commits or Semantic Commit Messages, or create your own rules that suit your project. The important thing is to be consistent and clear.

## How to edit or amend a commit message?

Sometimes, you may want to edit or amend a commit message after you have made it. For example, you may have made a typo, forgotten to include some details, or changed your mind about something. There are a few ways to do this in Git:

- If you want to edit the most recent commit message, you can use the command `git commit --amend`. This will open your default editor and allow you to modify the message. You can also use the `-m` option to specify a new message directly.
- If you want to edit an older commit message, you can use the command `git rebase -i HEAD~n`, where n is the number of commits you want to go back. This will open a list of commits in your editor and allow you to choose which ones you want to edit, reword, squash, or drop. **Be careful when using this command, as it can rewrite the history of your branch and cause conflicts with other branches**.
- If you want to edit a commit message that has already been pushed to a remote repository, you will need to force-push your changes with the command `git push --force`. However, **this is not recommended unless you are working on a solo project or a feature branch that no one else is using. Force-pushing can overwrite other people's work and cause confusion and frustration**.

## Examples of good and bad commit messages

To illustrate the difference between good and bad commit messages, let's look at some examples:

    Bad:
    git commit -m "done"

    Good:
    git commit -m "Implement user authentication feature"

---

    Bad:
    git commit -m "Fix bug"

    Good:
    git commit -m "Fix login error when username contains spaces"

---

    Bad:
    git commit -m "Update README"

    Good:
    git commit -m "Add installation instructions to README"

---

    Bad:
    git commit -m "Refactor code"

    Good:
    git commit -m "Refactor code to use async/await syntax"

---

    Bad:
    git commit -m "Add logging
    Logging is important for debugging and analysis"

    Good:
    git commit -m "Add logging

    Logging is important for debugging and analysis"

## Git history as a developer documentation tool

If you usually find yourself iterating over a project that has many similar parts that need to be implemented, such as doing a lot of simple CRUD APIs and GUIs for the same project, you will will find that following a pattern for your commits will help you a lot in being faster to write your code.

For example, if you're implementing an API controller, some business logic and some domain entities for a simple CRUD API, you may want to wrap the whole package in a single commit (if it is small and relevant enough to be understandable) and make it clear in the message that this commit represents this specific feature and all that is needed to make it work.

In this way, when implementing another similar feature, you can use this commit as a **guideline** on how to implement it and you'll be much faster in coding it.

Another aspect to consider is the **onboarding of new people to a project**. If you have a good enough Git history, a good documentation for how to setup your project will help your team mates setup the environment faster and the history will make them understand what the standard coding practices for the project are, without even needing to ask other experienced developers on the team.

## Conclusion

Writing clear and frequent commit messages is an essential skill for any developer who uses Git. It can help you and others understand what changes have been made and why they were made. It can also help you document your project and collaborate with other developers more effectively.

To write good commit messages, you should follow some general guidelines, such as capitalizing the first word, avoiding punctuation at the end, writing a short and descriptive title, writing an optional body with more details, using a blank line to separate the title and the body, and using a consistent style and format.

You should also commit frequently and in small packages of files, as this makes your commits more focused, granular, manageable, and mergeable.

You can edit or amend your commit messages if you need to, but be careful not to rewrite the history of your branch or overwrite other people's work.

You can also see the Git history of a project as a guideline to make onboarding of new team mates faster and as a way to document the process in a way that is close to the code itself.

By following these tips, you can write better Git commit messages that will make your life easier as a developer.

Programming, especially in a team, is all about **standards and expectations**. If you are consistent in your code writing, in your commit messages and in your workflow, you and your team mates will be very productive when reading each others code because you will know what to expect beforehand. You'll know for sure the pain that causes a codebase that is not well written and poorly (if not at all) documented.

Remember: in a few week (or in a few years!) you will have to go back to a feature or a bugfix you or your team mates implemented and you will thank yourself so much for having done a nice clean Git commit that everybody understands and is able to parse quickly.
