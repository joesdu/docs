### 完整步骤

1. **运行 `git filter-branch` 删除大文件**：

   ```sh
   git filter-branch --force --index-filter "git rm --cached --ignore-unmatch 1_src/setup/DeepLogic.Upgrade/about_us.mp4" --prune-empty --tag-name-filter cat -- --all
   ```

2. **清理和压缩存储库**：

   ```sh
   rm -rf .git/refs/original/
   git reflog expire --expire=now --all
   git gc --prune=now --aggressive
   ```

3. **强制推送到远程仓库**：

   ```sh
   git push origin --force --all
   git push origin --force --tags
   ```

### 解释

- **`--force`**：强制执行操作。
- **`--index-filter`**：在索引上执行过滤操作。
- **`"git rm --cached --ignore-unmatch 1_src/setup/DeepLogic.Upgrade/about_us.mp4"`**：删除指定文件。
- **`--prune-empty`**：删除空的提交。
- **`--tag-name-filter cat`**：保留标签。
- **`-- --all`**：应用于所有分支和标签。

通过这些步骤，你可以从 Git 历史记录中彻底删除大文件，并强制推送到远程仓库。确保在执行这些操作之前备份你的存储库，并通知你的团队成员，因为这些操作会重写历史记录。
