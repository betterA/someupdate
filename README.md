# someupdate

一个给 2–5 人小团队自用的 issue & agent 系统设计：**人只建 issue，剩下的 stage 由不同 agent 接力跑到 done。**
Python + SQLite，一个进程、一个文件，没有额外服务要装。

| 文件 | 是什么 |
|---|---|
| `DESIGN.md` | 完整设计（12 章）：三个概念分离、SQLite 表结构、多阶段回退、多 agent 路由、一致性、接口契约 |
| `DESIGN.html` | 同一份设计的图文版，浏览器直接打开 |
| `PROMPT.md` / `PROMPT-demo.html` | 生成这类 HTML 设计稿的风格规范与示例 |
| `to_agent.md` | 注入给 agent 的系统提示词（daemon 在 run 启动时拼进 prompt） |
| `datachange.md` | 一个 issue 从建单到 done 的全过程数据变化推演，末尾与设计逐条对照 |
