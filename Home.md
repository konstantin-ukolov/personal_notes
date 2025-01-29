Update Git Password:

To fix this on macOS, you can use

```
git config --global credential.helper osxkeychain
```

A username and password prompt will appear with your next Git action (pull, clone, push, etc.).
For Windows, it's the same command with a different argument:

```
git config --global credential.helper wincred
```

