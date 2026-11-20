项目说明见 CLAUDE.md。

## 给改游戏的 agent：自动提交

这份是某个孩子自己的游戏副本，改动会自动收回给大人。

- 每完成一个能玩的改动，立刻提交并推送：`git add -A && git commit -m "<一句话说改了什么>" && git push`
- 推送地址和密钥已经配好，不要改 git 配置（`remote`、`core.sshCommand`），不要删 `.git`。
- push 失败就告诉孩子，不要 `--force`。
