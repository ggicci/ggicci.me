---
title: "Open Source Contribution Workflow Best Practice"
date: 2019-01-08 00:48:13 +0800
categories:
  - git
tags:
  - git
  - github
  - open-source
---

Generally speaking, as we were using some open-source libraries we will inevitably find that there are some bugs or points that can be improved. And sometimes we even have to develop a brand new feature based on these libraries to satisfy our own needs. Out of respect for sprit of dedication, contributing our code to these opensource libraries is a good way to give back to the community.

## My Practice

When I decided to contribute to a repository, I will first try looking for if there is a contribution guide for it. Such guides usually lied as a section in the README or as a single document named CONTRIBUTING. Only when no guides were found, I will try to open a pull request to this repository on Github.

## Best Workflow

I'm using `github.com/gorilla/mux` as a sample repository to which I'm going to contribute. Here's my workflow:

1. On github, fork it to my github account as `github.com/ggicci/mux`.
2. Clone the repository to local: `git clone git@github.com:gorilla/mux.git`.
3. Rename remote `origin` to `upstream`: `git remote rename origin upstream`.
4. Add the URL of my forked repo as the `origin` remote: `git remote add origin git@github.com:ggicci/mux.git`.
5. Make some changes and commit: `git commit ...`.
6. Push the commits to my forked repo: `git push origin ...`.
7. On github, open a pull request from `ggicci` to `gorilla`.

## References

- [What is the difference between origin and upstream on GitHub?](https://stackoverflow.com/q/9257533)
