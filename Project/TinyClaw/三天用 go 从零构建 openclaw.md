## 第一天

理清框架，做一个最小的 mvp，先不实现 bus 功能（没有 bus 功能会违反 open/close design principle）
![[209289aebe5043b4406d61a3dd5ce692.jpg|500]]

## 第二天

实现 agent 模块

第一步先创建配置文件`config.go` 重要的函数是 `LoadConfig()`，使其能传递大模型所需的配置，apikey，model，baseurl

第二步创建 chatmodel，传入 config 

message --> chatmodel --> stream



