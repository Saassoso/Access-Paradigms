- List all markdown file in a repos 
```
curl -s https://api.github.com/repos/xxxx/git/trees/main?recursive=1 | grep '"path"' | grep '\.md' | sed 's/.*"path": "\(.*\)".*/\1/'
```