+++
date = 2025-08-25T07:10:59+02:00
title = "Why Bureaucracy is Good (Sometimes): The Case for Merge Requests"
description = "A discussion on why the overhead of merge requests, even for small changes, is a small price to pay for the immense benefits of team collaboration, visibility, and code quality."
authors = ["Stefano Previato"]
tags = ["software development", "git", "teamwork", "collaboration", "code review"]
categories = ["Career"]
+++

A colleague of mine recently brought up a point I've heard before, but it still makes me think. "Merge requests feel like too much bureaucracy," they said. We were looking at an issue that was just a quick text change on the frontend, and the thought process was, "Why do I have to create a new branch, a draft merge request, commit, and then un-draft it just for this tiny thing?"

I get it. The process can feel like a lot of steps, especially for a change that takes less than five minutes to implement. It can feel like putting on a full suit of armor just to walk across the street. . But here's the thing: that armor isn't just for show. It's a system of protection that ensures we, as a team, can move forward safely and efficiently without stepping on each other's toes.

### **The Grand Chessboard of Teamwork**

Think of our development workflow like a game of chess. Every piece, every move, needs to be intentional and visible to the rest of the team. When a developer pushes directly to our main `develop` branch, it's like a pawn suddenly teleporting to the other side of the board. The change is made, yes, but it’s done without visibility or a clear trail, potentially disrupting the entire game.

This is where the power of a standardized workflow comes in, and for us, that means **merge requests (MRs)**. A dedicated branch and an MR aren't just a list of tasks; they're a signal. They tell me, and the rest of the team, that a specific piece of work is underway. This prevents me from starting on an issue that you've almost finished, saving us all from wasted effort and a classic case of stepping on each other's feet.

Yes, for a small change, the overhead might seem like a lot. But is it really? We're talking about maybe two extra minutes of work per issue. In the grand scheme of things, that's a small price to pay for a clear, predictable workflow that saves hours of potential rework and confusion down the line.

### **More Than Just a Checkbox**

Beyond visibility, an MR is our central hub for collaboration. It's the dedicated space where we can:

- **Ask for reviews and comments.** This is a critical one for me. Even a small change can have unintended consequences. An MR provides a dedicated space for someone else to look at your work, offer suggestions, and catch potential bugs before they ever reach our testing or production environments. For the teammate who made the initial comment, this is something they haven't had to do yet, but for me, this workflow is the single most important part of my job.

- **See a linear history.** By using MRs, we can enforce a clean, understandable commit history. We can squash multiple small commits into one logical change when we merge. This ensures our `develop` branch doesn't look like a chaotic scribble, making it much easier to track down when and why a particular change was made.

- **Run and track tests.** Most modern tools like GitLab or GitHub integrate with our testing suites. The moment you open an MR, it can automatically kick off a test run. You can watch the progress, see what passes and what fails, and fix any issues directly within the context of that same MR. It's an incredible feedback loop that saves us from the headache of broken builds.

### **Practice Makes Perfect, and So Does an MR**

Even if you're working alone on a project, following this practice is a good habit to build. It's like a musician practicing with a metronome. It might feel a bit rigid at first, but it helps you develop the discipline and muscle memory you'll need when you join an orchestra. It ensures you are ready and seasoned when it's time to collaborate for real.

As AI tools become more integrated into our workflows, this becomes even more relevant. An MR is a perfect, clean space to collaborate with an AI agent. You can ask it to generate code or fix a bug, and you get to review its work in an isolated environment before it gets merged into the main codebase. It's the ultimate collaborative sandbox.

In the end, what may seem like an overhead is truly our secret sauce. It's the set of rules that lets us all play our best game without getting in each other's way. It’s the armor that protects our code base and our sanity. So while creating an MR for a single text change might feel like a lot, remember that it's a small investment in a robust and healthy development process that benefits everyone on the team.
