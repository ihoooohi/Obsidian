## Git 工作流规范

为保证代码质量与发布稳定性，项目采用基于分支的协作开发流程：

分支定义

• main：生产分支，必须始终保持可发布状态，禁止直接提交

• dev：开发主分支，用于集成功能并进行联调与测试

• feature/*：功能分支，每个功能或任务必须使用独立分支开发

• fix/*：缺陷修复分支，用于修复 bug

  

开发流程

1. 从 dev 创建分支

• 所有开发工作必须基于最新的 dev 分支创建：

  

git checkout dev

git pull

git checkout -b feature/<name>

  
  

2. 功能开发与提交

• 仅在 feature/* 或 fix/* 分支上进行开发

• 提交应保持小粒度且语义清晰

• 禁止直接在 dev 或 main 分支开发

3. 同步 dev 分支

• 在发起合并前，应同步最新 dev 并解决冲突：

  

git pull origin dev

  
  

4. 提交 Pull Request 到 dev

• 开发完成后，通过 PR 合并：feature/* → dev

• 必须经过 Code Review

• 必须通过必要测试（如 go test ./...）

5. 发布流程

• 当 dev 分支验证稳定后，通过 PR 合并：dev → main

• 该合并视为一次发布（Release）

  

约束要求

• 禁止直接向 main 分支提交代码

• 禁止绕过 PR 流程直接合并分支

• 每个分支只承载单一功能或修复，避免混合变更

• 分支应保持短生命周期，合并后及时删除

• 所有变更必须通过 PR 记录，确保可追溯