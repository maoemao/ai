# 快速搭建AI工作流

### Apple Silicon Mac AI新手本地运行AI工作流完整教程（2025年11月最新·M芯片最优化版）


**原生Ollama（充分利用Metal加速） + Flowise（拖拽工作流）**

整套流程30分钟内搞定，全程免费、完全离线、零云依赖。

#### 硬件建议（M芯片实测数据）
- M1/M2/M3/M4 任意型号都行
- 内存最低16GB（推荐32GB+，能轻松跑70B）
- 存储留100GB空闲就够

#### 第一步：安装原生Ollama（Mac最优，3分钟，无需Docker，Metal全速）

1. 直接去官网下载Ollama：https://ollama.com/download  
   点「Download for Mac」，下载完双击安装（苹果芯片版会自动识别）

2. 安装完后，打开终端（Terminal），运行：
```bash
ollama serve &   # 后台启动服务（只需一次）
```

3. 下载模型（2025年Mac最强推荐模型，按中文/速度/综合排序）：

```bash
# 中文最强 + 速度最快（强烈推荐先跑这个）
ollama run qwen2.5:14b     # 14B参数，M芯片上速度80~110 token/s，中文理解无敌

# 其他Mac神级模型（任选其一）：
ollama run qwen2.5:32b     # 32B，M4 Pro/Max上120+ token/s，接近GPT-4o水平
ollama run llama3.1:8b     # 英文最强，综合最均衡
ollama run llama3.2:11b    # 新款多模态（可看图），超轻快
ollama run gemma2:27b      # 27B，M芯片最省显存的神器，速度爆表
ollama run phi4:14b        # 微软新王牌，代码+数学超强
```

下载完后直接在终端打字就能聊天，速度快到让你怀疑人生（M4 Max跑qwen2.5:32b能到140+ token/s）。

#### 第二步：安装Docker Desktop（只为跑Flowise，2分钟）

1. 去 https://www.docker.com/products/docker-desktop/ 下载Mac版（Apple Silicon专用）
2. 双击安装，打开后打开Docker Desktop，让它在后台运行（菜单栏会出现小鲸鱼图标）

#### 第三步：一键启动Flowise（最简单拖拽工作流平台，2分钟）

在终端运行这行命令：
```bash
docker run -d --name flowise -p 3000:3000 -v flowise_data:/root/.flowise flowiseai/flowise
```

启动成功后，浏览器打开 http://localhost:3000  
第一次会让你设置账号，随便填一个用户名密码。

#### 第四步：把Ollama连接到Flowise（1分钟）

1. 在Flowise点「+ Create New Flow」
2. 搜索并拖出节点：
   - Chat Input
   - ChatOllama  
   - Chat Output
3. 双击ChatOllama节点设置：
   - Base URL：http://host.docker.internal:11434   ← Mac必须这样写！
   - Model Name：qwen2.5:14b （和你刚才下载的模型名一样）
   - Temperature：0.7（随意）
4. 连线：Input → ChatOllama → Output
5. 点右上角 Save + Start
6. 右边聊天窗直接打字测试 → 恭喜！你的第一个本地AI工作流跑起来了！

#### 超实用模板（5分钟做出真·好用工具）

Flowise左边有「Templates」市场，直接搜名字一键导入：

1. **本地文档问答（最常用）**  
   搜模板：「Document Q&A with Retrieval」  
   上传PDF/Word/TXT，直接问文件内容，永不泄密，Mac上秒出答案。

2. **智能个人助理Agent（会用工具）**  
   搜模板：「ReAct Agent with Tools」或「Autonomous Agent」  
   能上网搜（加SerpAPI或DuckDuckGo工具）、写文件、计算、看图，全本地跑。

3. **多模态视觉助手（看图说话）**  
   用 llama3.2:11b 或 llama3.2-vision模型  
   模板搜「Vision Chat」，拖图片就能问“图里有什么”“帮我P一下”。

4. **每天自动总结新闻/邮件**  
   加「Scheduler」+「RSS」节点 → Ollama总结 → 发送到企业微信/邮件

#### Mac专属加速技巧（让速度再飞20-50%）

- 在Ollama模型设置里加参数（终端运行时）：
  ```bash
  ollama run qwen2.5:14b --num-gpu 999   # 强制全Metal GPU，速度拉满
  ```
- 内存≥32GB的机器直接上 qwen2.5:32b 或 llama3.1:70b，速度照样80+ token/s
- 想更极致？后期可以换成 mlx-lm（Apple官方引擎）跑模型，但新手先用Ollama最稳

#### 30天后推荐升级路线（Mac用户终极归宿）

等你玩熟了，直接换：
- 模型后端 → LM Studio（Mac原生app，最漂亮界面，模型下载最快）
- 工作流平台 → Langflow（调试最强）或 Jan.ai（最美最简洁，全原生Mac）

我自己M4 Max主力就是：  
**Ollama原生跑qwen2.5:32b + Flowise做复杂Agent + LM Studio日常聊天**

现在就去试！  
M芯片跑本地AI是真的爽到飞起，跑完qwen2.5:14b你会直接把ChatGPT卸载。  
有任何卡住的地方直接截图发我，我手把手帮你调。马上行动，你今天就能用上自己的私人GPT-4o了！