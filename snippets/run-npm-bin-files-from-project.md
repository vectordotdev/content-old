Many NPM packages come with binaries. Unless they're installed with the `--global` flag, they won't be available on the system path by default. You can use NPM scripts, but sometimes you just need to manually call the binary (to see help menus, for example) and don't want to install as a global.

###### .bashrc / .zshrc
```bash
# ...

npm-run() {
  $(npm bin)/$*
}
```

The following command will now use `./node_modules/.bin/gitdocs` local to the project, rather than needing a global installation.

```bash
npm-run gitdocs serve
```
