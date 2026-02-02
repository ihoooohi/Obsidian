下面按你提供的原文，**逐字逐句（尽量逐句对照）**翻译成中文；我保留原来的标题、项目符号、编号、代码/接口片段、表格与注释格式。

---

## Version 1 Walkthrough Overview:

## --------------------------------

## **Version 1 讲解式流程概览：**

在 README 的末尾，我们让你思考你可能需要哪些类（classes）。现在我们要用一种“边想边说（think aloud）”的方式来做同样的练习。也就是说，这里写下的内容是：先有我最初的想法，然后是对它的思考、修改、修订等。与其给你一个打磨得很漂亮的最终成品，我更希望你能看到尽可能多的、用文本能够表达清楚的思考过程。

我从描述中挑出了一些对我来说很显眼的类作为起点：

- 类：Board（棋盘/板）
    
    - 字段：width + height（宽度 + 高度）
        
    - 字段：ships（船）（列表？从坐标到船的映射？集合？）
        
    - 方法：检查船只放置是否合法（check validity of ship placement）
        
        - 这听起来足够复杂，应该把它抽象出来
            
    - 方法：放置棋子/船（place piece）
        
    - 方法：检查某个格子里是什么（check what is in a square）
        
        - 这会以多种方式发生：命中/未命中、显示“自己的”海域、显示“敌方”的海域
            
    - 方法：检查胜利/失败（check for winning/losing）
        
    - 方法：displayBoard（显示棋盘）
        
        - 需要一个参数，表示显示自己的还是敌人的
            
- 类：Ship（船）
    
    - 字段：在棋盘上的坐标（coordinates on board）
        
    - 字段：哪些格子被击中（which squares have been hit）
        
    - 字段：船类型的字母（s, d, c, b）
        
    - 字段：船类型名称的字符串（string for the name of the ship type）
        
    - 方法：isSunk（是否沉没）
        
    - 方法：getletter（获取字母）
        
    - 方法：occupiesCoordinates（是否占据某坐标）
        
    - 方法：getname（获取名称）
        
- 类：InputParser（输入解析器）
    
    - 方法：readAndParsePlacement //例如 A0V 或 M3H
        
    - 方法：readAndParseTarget //B2 或 H8
        

从列出这些后，我意识到我真的很希望有 Coordinate（坐标）和 Placement（放置）这两个类。把它们加进去后，我们得到了我称之为“551 级”的面向对象设计：我们已经识别出了要变成类的名词，以及与它们配套的动词。我们把大问题拆成了几个部分，让整体更易管理。这种设计水平是我期望你在学期开始时能做到的，但我们现在需要对它进行**大幅改进**。

下面是我对最初想法的几条批评（问题点）：

- Board 没有遵守 SRP（单一职责原则）。想想它目前被绑定到了哪些需求：
    
    - 胜/负：如果我们改变胜负判定规则怎么办？我们就需要修改 Board
        
    - 船只放置合法性：我们可能也会改变这些规则。之后我们可能决定船之间需要保持最小距离等。这也会迫使我们修改 Board
        
    - 显示棋盘（包括显示自己/敌人）  
        如果我们改变显示方式呢？如果我们做 GUI 呢？
        
    - 跟踪棋盘状态
        
- Ship 和 Board 都与我们的文本表示（textual representation）紧耦合（这也意味着它们违反 MVC）。
    
    - 对 Board 来说，如果是 GUI，我们不会想把它 print 出来，而是要把它画出来（draw）
        
    - 对 Ship 来说，在 GUI 中我们不会想要一个用来显示的字母，我们可能想要颜色或图片
        
- InputParser 里的方法把“读取输入”和“解析输入”紧耦合在一起（注意方法名里的 “and”）。  
    我们可以把它们拆成 InputParser 里的多个方法，但我们也可以观察到：我们可以让 Coordinate 和 Placement 拥有接收 String 的构造函数。
    
- 我们依赖的是具体类，而不是接口（interfaces）
    

在继续之前，我们先花点时间处理这些问题：

---

## 1. 用泛型降低 Ship/Board 与文本界面的耦合

1. 首先，让我们看看 Ship + Board 与基于字符的文本界面之间的紧耦合。为了达成我们的目标，这些类里确实需要一些数据（例如一个字符），但我们不想把这种假设“写死（bake in）”。相反，我们希望以后在数据类型上保持灵活。这正是泛型（Generics）的绝佳用法。
    

我们可以做一个 Ship，其中 T 是视图（view）需要的信息类型。对于文本版战舰，我们会用 Ship。如果你以后加 GUI，你可以用 Ship，甚至 Ship。

那么我们是否只让 Ship 持有一个 Character（例如 's'）作为显示信息就够了？我们至少需要两个：一个表示类型（例如 's'），另一个表示命中时要显示什么（例如 '*'）。

对于 GUI 实现，我们甚至可能需要更复杂的东西：我们可能对船的每个“片段（piece）”使用不同图片（船头、中间、船尾），并且对被击中或未击中的片段使用不同图片。

于是我们可以设想一个接口：ShipPieceDisplayInfo（这里原文后面给的是 ShipDisplayInfo）

```java
interface ShipDisplayInfo<T> {
   public T getInfo(Coordinate where, boolean hit);
}
```

然后我们的 Ship 可能会有：

```java
public class Ship<T> {
   private ShipDisplayInfo<T> displayInfo; // 在构造函数中注入

   public T getDisplayInfoAt(Coordinate c) {
     // 代码稍后再写
   }
}
```

---

## 2. 重构 Board：分离视图、分离规则、分离胜负逻辑

2. 接下来，我们处理 Board。以下是我们一开始的版本：
    

- 字段：width + height
    
- 字段：ships（列表？从坐标到船的映射？集合？）
    
- 方法：检查船只放置是否合法
    
    - 这听起来足够复杂，应该抽象出来
        
- 方法：place piece（放置棋子/船）
    
- 方法：检查某格子里是什么
    
    - 发生在多种方式：命中/未命中、显示自己的海域、显示敌人的海域
        
- 方法：检查胜/负
    
- 方法：displayBoard（显示棋盘）
    
    - 带参数决定显示自己还是敌人
        
- 第一，把 displayBoard（视图 view）从 Board（模型 model）中分离出来。我们将有一个 BoardTextView 类。并且，当我们把 Ship 改成 Ship 后，我们的 Board 也需要变成 Board。
    
- 第二，检查放置合法性是我们很希望 Board 能做的事情……但我们也想把它分离出去。我们能不能两全其美？当然可以！
    

我们可以做一个 PlacementRuleChecker（放置规则检查器）的接口，然后让 Board 的合法性检查像这样：

```java
public boolean canPlace(Ship piece,
                        Placement where,
                        PlacementRuleChecker checker)
```

你可能觉得 PlacementRuleChecker 会很难看（很糟糕），但我们稍后会回到这里，并看到一个非常漂亮的做法！

- 第三，我们可以把胜/负逻辑单独抽出来放进一个类。我们给它做成接口（叫 CompletionRules），这样以后就能很容易改变规则。
    

---

## 3. 让 Board/Ship 依赖接口而不是具体类；思考船的具体实现

3. 我们目前依赖的是具体类（Board 和 Ship），而不是接口。让我们把 Board 做成接口，并用 BattleShipBoard 作为具体实现。同样，我们也把 Ship 做成接口。那具体类有哪些呢？
    

一种想法可能是：

- CarrierShip（航母船）
    
- DestroyShip（驱逐舰船）
    
- SubmarineShip（潜艇船）
    
- BattleShip（战列舰）
    

但请注意：这些类实际上只在 _数据_ 上不同，而不是行为。例如，一个是 1x2 且字母是 's'，另一个是 1x4 且字母是 'b'。

我们也可以想象：

- RectangleShip（矩形船）
    

它可以覆盖 Version 1 的所有船。然后我们可以为 Version 2 的船再写其他类。

我们还可以意识到：我们所有的船其实都只是“坐标集合”（再加上一些相关行为）。这意味着我们可以有这样的继承层级：

```
interface Ship
 |
 -- abstract class BasicShip // 一组坐标
          |
          |-- RectangleShip  // 覆盖所有 Version 1 船
          |-- TShapedShip    // Version 2 的战列舰
          |-- ZShapedShip    // Version 2 的航母
```

好，_现在_ 我们对类的样子有一个很好的概念了：

1. interface Board
    
2. class BattleShipBoard implements Board
    
3. interface Ship
    
4. abstract class BasicShip implements Ship
    
5. class RectangleShip extends BasicShip
    
    - 在 version 2：class TShapedShip extends BasicShip
        
    - 在 version 2：class ZShapedShip extends BasicShip
        
6. class Coordinate
    
    - 有 int row + int column
        
7. class Placement
    
    - 有 Coordinate where +（某种东西？）orientation（方向）
        
8. class BoardTextView
    
9. interface ShipDisplayInfo
    
10. interface PlacementRuleChecker
    
11. interface CompletionRules
    
12. class App（顶层类：把所有东西组装起来）
    

我们还有几件事需要思考：  
(1) 我们不是只显示一张棋盘，而是两张：这应该放在 BoardTextView 里还是另一个类里？  
(2) 在上面 [9]（这里原文写 [9]，但它指的是我们 Placement 类那项）里，我们对 orientation 写了 ???。我们想用什么？  
用 char（Version 1 的 H/V，Version 2 的 H/V/U/D/L/R）是可行的，但不舒服。  
我们会先用 char 来表示方向（即使我们不喜欢），然后看看效果如何。  
(3) 我还没把所有类的所有细节都写出来。没关系。我们已经有了大图景，并且可以开始做第一批任务了。

---

## 接下来：任务拆分、顺序与时间估算

接下来我们试着想想我们的任务有哪些、按什么顺序做、以及大概会花多久。注意：所有任务都包含该任务的测试用例与代码注释。不过，这些是开发时间估计（就好像你只是写代码）。我把估计调整成我对你们的预期，而不是我对我自己的预期。这些估计**不包括**你阅读详细说明所花的时间——阅读会让任务多花一点时间。

我也把它们分成 5 个主要“目标（goals）”，每个目标再拆成任务。你的 extra credit 将是：在指定日期前交付每个目标。

---

### //Goal 1: minimal end to end system（最小端到端系统）

```
任务                                             时间估计 [~2 小时]
------------------------------------------------+------------------------
0. 项目设置（Project Setup）                           5 分钟
1. BattleShipBoard 类                                10 分钟
   - 暂时只做 width/height
   - 以及 getters/setters
2. BoardTextView（只显示空棋盘）                       15 分钟
3. Coordinate 类                                      10 分钟
   - 包括接收 "A0" 字符串的构造函数
4. Placement 类                                       10 分钟
   - 包括接收 "A0V" 字符串的构造函数
5. Ship 接口                                          5 分钟
6. 占位版 BasicShip 类                                15 分钟
   - 暂时还不是抽象类
   - 只有一个 Coordinate
   - 大多数方法先硬编码
7. 把船加入 BattleShipBoard                           30 分钟
   - 添加字段来保存它们
   - 添加 place 方法
   - 暂时不检查放置合法性
   - 获取某格子里是什么
       - 只针对自己的棋盘（不是敌方棋盘）
       - 不跟踪这个格子是否曾“未命中”
8. 更新 BoardTextView：能显示带船的棋盘               15 分钟
9. App 骨架（skeleton）                               15 分钟
   - 创建一张棋盘
   - 读取一个放置（暂时无错误处理）
   - 把船放到棋盘上
   - 显示棋盘
   - 退出
```

注意：第一个目标从非常少的状态开始（我们能做出的最简单的 “Board”），并逐步构建到一个端到端系统。这个最小端到端系统拥有我们需要的“骨架”：它读取一个放置（只放一艘船），把那艘船放到棋盘上（现在只是单格占位符），然后显示棋盘。我们暂时**不做**无效放置等错误处理。

现在我们要开始去掉占位符并添加功能。首先让我们做真正的船：

---

### //Goal 2: real BasicShip class and RectangleShip（真正的 BasicShip 与 RectangleShip）

```
任务                                             时间估计 [~1.5 小时]
------------------------------------------------+------------------------
10. 实现 RectangleShip                                 30 分钟
    - BasicShip 现在变为抽象类
    - RectangleShip 有 width/height
11. 在 BasicShip 中加入“命中跟踪（hit tracking）”        30 分钟
12. 在 BasicShip 中加入名字与显示信息（display info）     30 分钟
```

---

### //Goal 3: check for valid placements（检查放置合法性）

```
任务                                             时间估计 [~1 小时]
------------------------------------------------+------------------------
13. 边界内规则（In Bounds Rule）                       20 分钟
    - 确保新放置不会超出棋盘
14. 无碰撞规则（No Collision Rule）                     20 分钟
    - 确保新放置不会与已有船重叠
15. 把它们整合起来（Put it all together）               20 分钟
```

---

### //Goal 4: finish up "placement phase" of game（完成“放置阶段”）

```
任务                                             时间估计 [~1.5 小时]
------------------------------------------------+------------------------
16. 放置所有船（Place all ships）                      30 分钟
17. 放置的错误处理（Error handling for placement）     30 分钟
18. 第二位玩家（Second Player）                        30 分钟
    - 你需要先做一个玩家的放置，再做另一个。
      我们必须在这里加上这一点
```

---

### //Goal 5: "Attacking Phase" of game（“攻击阶段”）

```
任务                                             时间估计 [~2 小时]
------------------------------------------------+------------------------
19. 显示敌方棋盘（Display board for enemy）            30 分钟
20. 并排显示两张棋盘（Display two boards side by side）30 分钟
21. 检查胜/负（Checking for win/lose）                 30 分钟
22. 与用户交互（Interact with user）                   30 分钟
    - 读取坐标（+ 错误处理）
    - 显示结果
```

这些估算合计大约是 version 1 的 8 小时。考虑到你还要阅读详细说明等，我们把它增加到 10–12 小时。我们也**强烈**建议你尽早开始。不要等到截止前 12 小时才试图用一次马拉松完成。相反，我们建议如下安排：

- 作业发布：1 月 15 日（Jan 15）
    
- 在 1 月 19 日 11:59 PM 前完成 Goal 1
    
- 在 1 月 22 日 11:59 PM 前完成 Goal 2
    
- 在 1 月 24 日 11:59 PM 前完成 Goal 3
    
- 在 1 月 26 日 11:59 PM 前完成 Goal 4
    
- 在 1 月 28 日 11:59 PM 前完成 Goal 5
    
- Version 2：在 1 月 28 日到 2 月 5 日之间完成
    
    - 你将为 Version 2 做自己的设计与实现
        
    - 我们建议你把 Version 2 拆成任务，并为中间节点设定自我截止时间
        
- 作业截止：2 月 5 日 11:59 PM（Feb 5）
    

对以上至少每个任务都进行 git commit 与 git push（因此 version 1 你至少会有 22 次 git commit 和 git push）。注意：正如我们稍后课程会讲到的，当你进行团队协作时，你需要经常合并代码。现在就养成频繁提交与推送的习惯是好的。另外，git commit 与 git push 也是我们评分时会考虑的因素之一。

在继续之前，我们想强调一下：这种任务拆分把一个看起来非常吓人的大任务（“我要怎么写战舰？？？”）变成了 22 个规模更可控、每个都更容易上手的任务。

现在我们已经思考了类设计，并把事情拆成任务，我们就准备开始了。我们将带你完成这些任务：通常在开始时给更多细节，到后面给更少细节。

本 walkthrough 的剩余部分被拆分成对应各个任务的文件（task0.txt、task1.txt、……），并且每个文件放在与其对应目标（goal）的目录中。当你准备继续时，你应该去 goal1/task0.txt，并从那里开始。

---

如果你还想要“英文一句 + 中文一句”完全对照排版版（更适合背诵/核对），我也可以把这份再改成逐行对照格式。