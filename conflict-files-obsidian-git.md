# Conflicts
Please resolve them and commit them using the commands `Git: Commit all changes` followed by `Git: Push`
(This file will automatically be deleted before commit)
[[#Additional Instructions]] available below file list

- Not a file: .obsidian/app.json
- Not a file: .obsidian/graph.json
- Not a file: .obsidian/plugins/obsidian-git/data.json
- Not a file: .obsidian/plugins/obsidian-git/main.js
- Not a file: .obsidian/plugins/obsidian-git/manifest.json
- Not a file: .obsidian/workspace.json
- [[무중단 배포 가이드]]

# Additional Instructions
I strongly recommend to use "Source mode" for viewing the conflicted files. For simple conflicts, in each file listed above replace every occurrence of the following text blocks with the desired text.

```diff
<<<<<<< HEAD
    File changes in local repository
=======
    File changes in remote repository
>>>>>>> origin/main
```