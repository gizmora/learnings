
### Semantics Message commits

1. feat: `commit message`
2. docs: `commit message`
3. chore: `commit message`
4. fix: `commit message`
5. refactor: `commit message`
6. style: `commit message`
7. test: `commit message`
8. localize: `commit message`

chore: add Oyster build script  
docs: explain hat wobble  
feat: add beta sequence  
fix: remove broken confirmation message  
refactor: share logic between 4d3d3d3 and flarhgunnstow  
style: convert tabs to spaces  
test: ensure Tayne retains clothing 


### Submodules

1. **Updating root branch after update in submodule**

```
git add frontend backend
git commit -m "Update frontend & backend submodules"
git push origin master
```

2. **Cloning a repo with submodules**

```
git clone <deploy_repo_url>
cd deploy
git submodule update --init --recursive
```

3. **Pulling update**

```
git pull origin master
git submodule update --init --recursive --remote
```

4. **Removing submodule**

```
git submodule deinit -f frontend
git rm -f frontend
rm -rf .git/modules/frontend
git commit -m "Remove frontend submodule"
```

