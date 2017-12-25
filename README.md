# Easystack Contain OS Docs

Documentation site for Easystack Contain OS opensource projects.

## Build

To view the documentation site locally, you need to clone this repository:

```
git clone -b hugo git@github.com:easystack-cos/easystack-cos.github.io
cd yakDocs
git worktree add -B gh-pages public origin/gh-pages
git worktree add -B master content origin/master
git submodule init
git submodule update
```

Then to view the docs in your browser, run Hugo and open up the link:

```
$ hugo server

Started building sites ...
.
.
Serving pages from memory
Web Server is available at http://localhost:1313/ (bind address 127.0.0.1)
Press Ctrl+C to stop
```
