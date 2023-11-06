
 [https://ayedaemon.github.io](https://ayedaemon.github.io/)


---


- Run locally

```
# Clone repo
git clone <url_to_repo>

# Start hugo server

hugo server -D --disableFastRender
```


- Create new post page


```
hugo new path/to/new/file.md
```


- Deploy (now using worktrees)

0. One time setup (initial)

```
git worktree add -B gh-pages public origin/gh-pages
```


1. Build hugo site

```
hugo --gc --minify --cleanDestinationDir --logLevel info
```

2. Push main branch

```
git push
```

3. push gh-pages branch

```
cd public && \
git push &&\
cd -
```
