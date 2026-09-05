# Day32 - Apache POI及项目收尾

## 一、Apache POI

可以使用POI再Java中对office文件进行读写操作

### 1.1 写操作

- XSSWorkBook创建excel文件
- excel.createSheet()创建sheet对象
- sheet.createRow()创建row对象
- row.createCell().setCellValue()

- 通过输出流将内存中的excle写入到磁盘中
- FileOutPutStream

- 最后要关闭资源
- out.close()
- excel.close

### 1.2 读操作

- 通过输入流读取磁盘文件
- 再通过有参构造封装为excle对象

项目结尾

对于项目的一些问题、思考及回答

---

## 二、第一层：架构与基础设施

### Q1 模块拆分的目的

**问题**：三个模块为什么要这样拆？举个具体例子说明：如果订单 DTO 放在 sky-common 会有什么问题？

**你的回答**：

> 将项目拆成 common、pojo、service 三个模块能有效提高代码可读性以及可维护性，也是在项目模块上的一个低耦合高内聚思想的实现。如果订单 DTO 放在 common 模块，在查找想要的这个类时就不会很方便，需要花时间去寻找这个 DTO 放在了哪个模块的哪个包下；同时，每一个模块所使用的 pom 依赖也会不同，可能会引起一些依赖上的问题。

**点评 / 标准答案**：

- 三个模块的定位：common = 与业务无关的通用设施（Result、异常体系、常量、JWT/OSS/微信支付工具）；pojo = 纯数据载体；server = 业务实现。
- **真正原因是依赖方向，不是"好不好找"**：`server` 依赖 `common` 和 `pojo`，且依赖是单向的；common 永远不能反过来依赖业务类型。
- 订单 DTO 放 common 的问题：① common 被业务污染，不再"通用"；② 可能造成"基础模块依赖业务模块"的反向依赖，破坏整个依赖体系。
- 对"pom 依赖不同"说法的修正：拆模块恰恰是为了让依赖可控（pojo 不需要 Spring Web / MyBatis，common 只引必要依赖），DTO 放错位置本身不会引发依赖冲突，破坏的是职责边界。

### Q2 VO / DTO 的区别与模块放置

**问题**：你项目里 VO 和 DTO 分别什么时候用？举个具体例子：为什么用户下单的入参用 OrdersSubmitDTO，而给前端展示订单详情用 OrderVO，不能直接拿 Orders 实体返回？另外：BaseContext、JwtClaimsConstant、AutoFillConstant 分别在哪个模块？为什么它们适合待在 common？

**你的回答（第一问，回答中断）**：

> DTO 用于接收浏览器传过来的参数，VO 负责封装返回浏览器的参数。因为 DTO……（回答中断）

**你的回答（第二问）**：

> 这三个都放在 common 模块，分别放在 context 包和 constant 包下，因为它们被 server 模块所引用。BaseContext 是 JWT 和一些 service 中 getUserId 所需要的依赖，这些常量也是业务模块所需要的依赖。

**点评 / 标准答案**：

- **Entity 对数据库，DTO 对请求，VO 对响应**，三者职责分离。
- 不直接返回实体的三个理由：
  1. **防泄露**：实体字段对应表结构，直接返回会把内部字段（openid、备注等）暴露给前端；
  2. **装不下组合信息**：订单详情需要 `List<OrderDetail>`、拼接好的 `orderDishes` 字符串，实体放不下；
  3. **解耦**：前端接口结构不被表结构绑架，改表不影响接口。
- 三个类的归属判断对了，但"被 server 引用"不是充分条件（controller、mapper 也被引用，就不在 common）。真正的判断标准：
  - **与具体业务无关**、属于通用机制；
  - 会被多处/公共模块自身复用（JwtUtil、JwtProperties、JwtClaimsConstant 属于同一套认证机制）；
  - **依赖方向允许**——common 不能依赖业务类型。

---

## 三、第二层：核心业务

### Q3 购物车合并逻辑

**问题**：购物车添加商品时，同菜品且同口味的条目怎么做数量合并？口味不同的菜品又是怎么处理的？删除/清空购物车的 key 分别是什么？

**你的回答**：

> 对于同一个菜品同一种口味加两次，是合并为一条、数量翻倍，是判断 setmealId 是否相同；同一菜品口味不同就是单独的两条数据记录。最后清空购物车是清空当前 userId 的购物车。

**点评 / 标准答案**：

- 判定条件在 `ShoppingCartMapper.xml` 的 `list` 动态 SQL，共四个条件：`user_id`、`dish_id`、`setmeal_id`、`dish_flavor`。
- **菜品**：`user_id + dish_id + dish_flavor` 都相同才合并；同一菜品口味不同 → dish_flavor 不同 → 新增一条。
- **套餐**：`user_id + setmeal_id` 相同即可合并（套餐没有口味）。
- 查不到就新增，并回查菜品/套餐表补 `name / image / amount`，`number` 置 1。
- 清空：`clean(BaseContext.getCurrentId())`，按当前登录用户删整条购物车，下单成功后也调用它。
- 细节：`sub`（减一）时数量 > 1 则 `number - 1` 更新，数量 = 1 则 `deleteById` 删除记录。

### Q4 下单流程与订单号

**问题**：从用户点"提交订单"开始，到购物车被清空，完整流程讲一遍（接口 → 校验了什么 → 插入了哪几张表 → 什么时候清购物车 → 返回了什么）。其中订单号是 `String.valueOf(System.currentTimeMillis())` 生成的，高并发下会有什么问题？如果让你改进，你会怎么生成订单号？

**追问**：下单时 orders 表里要写 phone（电话）、address（地址）、consignee（收货人），这三个字段是从哪来的？再想想 orders.amount（订单金额）——它是后端根据购物车重新算的，还是直接用了前端传来的值？如果是后者，这里有什么风险？

**你的回答**：

> 用户点击提交订单，向服务器端发送请求，此时校验 JWT 令牌是否正确；然后请求发送到对应的提交订单接口，由 controller 层用 DTO 封装好请求参数，调用 service 的提交订单方法。首先处理异常情况，从线程中获取 userId，根据 userId 调用 mapper 层查询当前 userId 的购物车是否有数据，判断传输过来的订单是否为空，并用地址 id 判断用户地址是否为空；然后利用 userId 调用 mapper 查询购物车信息，将信息插入到订单表和订单明细表中；所有信息插入完毕，就调用清空购物车的方法。因为这个 service 同时处理了多张表，所以要加上事务注解保持数据一致性。高并发下可能出现订单号重复的情况，我可能会使用时间戳加加盐加密操作生成订单号。

**你的回答（追问）**：

> 这三个字段（phone、address、consignee）都是从前端的请求数据中传输过来的，通过 DTO 封装后在 service 层将数据插入到 orders 表中。这个金额好像是从前端传过来的吧，我记得我好像没有处理这个值。可能会有人直接修改前端数据欺骗服务器，然后传输过来一个虚假的值，造成我的收入损失，并不安全。

**点评 / 标准答案**：

- 流程正确：校验地址簿为空（`AddressBookBusinessException`）→ 校验购物车为空（`ShoppingCartBusinessException`）→ 插 `orders` → 插 `order_detail`（批量）→ 清空购物车 → 返回 `OrderSubmitVO`。`@Transactional` 保证多表一致。
- **订单号**："加盐加密"解决的是不可预测/防篡改，**不解决唯一性**。标准方案：
  - UUID：简单唯一，但 36 位太长且无序（微信商户订单号有长度限制）；
  - **雪花算法（Snowflake）**：时间戳 + 机器号 + 序列号，唯一且趋势递增，约 19 位；
  - Redis INCR / 数据库序列：如 `20260827 + 000001`，可读性好。
  - 兜底：订单号字段建**唯一索引**，防止极端情况重复。
- **安全细节（重点）**：`phone / address / consignee` **不是前端传的**，是从地址簿表查出来的（前端只传 `addressBookId`），这是安全设计；而 `amount` **确实来自前端**（DTO 有 amount 字段，BeanUtils 直接拷贝），是真实安全漏洞——正确做法是后端根据购物车明细重算金额，支付金额以后端为准，回调里还要校验金额一致。

### Q5 订单状态机

**问题**：订单的 1~6 状态分别是什么？用户取消、商家拒单、商家取消、派送、完成，各自要求订单处于什么状态才合法？其中"商家拒单"和"用户取消"如果订单已经支付了，除了改状态还必须做什么？

**你的回答**：

> 1 待付款、2 待接单、3 已接单、4 派送中、5 已完成、6 已取消。6、6、6、4、5（操作后的状态）。还要将已支付的钱通过微信支付接口将金额返还。

**点评 / 标准答案**（题目问的是**前置状态**，你答成了"操作后变成几"）：

| 操作 | 前置状态（必须） | 目标状态 |
| --- | --- | --- |
| 用户取消 | 1 或 2（代码：`status > 2` 抛异常） | 6 已取消 |
| 商家拒单 | 2 待接单 | 6 已取消 |
| 商家取消 | 代码中**无校验**（可改进点） | 6 已取消 |
| 派送 | 3 已接单 | 4 派送中 |
| 完成 | 4 派送中 | 5 已完成 |

- 退款：已支付订单取消/拒单必须调用微信退款接口并更新 `payStatus`；本项目测试环境退款已注释、用日志模拟。小瑕疵：`userCancelById` 会更新 `payStatus = REFUND`，但商家拒单和商家取消没有更新。
- 记忆口诀：**状态流转是链式的，只能从紧邻的前置状态跳过来**；1 和 2 可以随时取消成 6，跳步即非法。

### Q6 缓存实现（Redis）

**问题**：菜品和套餐的缓存分别是怎么做的？为什么查询套餐要按分类 id 分别缓存（`setmealCache_{分类id}` 这种 key）？增、删、改菜品和套餐时，分别在哪些地方清除了哪些缓存？有没有哪一步是"容易漏清"的？

**你的回答**：

> 具体的我不记得了，但是我知道都是通过 Redis 来实现。然后有一个是直接在业务逻辑上加判断，是否需要加入缓存、删除缓存；另一个是基于 Spring Cache 提供的注解方式，直接在方法上添加注解来判断是否删除缓存、加入缓存。使用这种方式让 key 更便于阅读，同时避免了 key 重复。添加菜品或套餐就清除菜品的缓存，删除菜品需要清除菜品的缓存，删除套餐需要清除套餐的缓存，修改菜品需要清除菜品和套餐的缓存，修改套餐需要清除套餐的缓存。

**点评 / 标准答案**（本项目真实实现，你的"手动/注解"对应关系记反了）：

- **菜品缓存 = 手动 Redis（写在 controller 层）**：
  - 用户端查询：`key = "dish_" + categoryId`，先查 Redis，命中直接返回，未命中查库后写入；
  - 管理端清理：新增 → 清 `dish_{该分类id}`；批量删除 / 修改 / 起售停售 → 清 `dish_*`（`keys` 模式匹配后删除）。
- **套餐缓存 = Spring Cache 注解**：
  - 用户端：`@Cacheable(cacheNames = "setmealCache", key = "#categoryId")`；
  - 管理端：新增 → `@CacheEvict(key = "#setmealDTO.categoryId")` 只清本分类；删除 / 修改 / 起售停售 → `allEntries = true` 全清。
- 优缺点：手动 Redis 直观可控但样板代码多，`keys dish_*` 在大数据量下是 O(N) 全库扫描（生产用 SCAN 或精确删）；Spring Cache 声明式一行搞定，但可配置性弱，且注解加在 controller 上，绕过 controller 调 service 不生效。
- **容易漏清的点**：菜品停售时级联把包含它的套餐也停售（`updateStatus`），但只清了 `dish_*`，套餐缓存没清；目前套餐缓存不含菜品明细所以影响不大，但这是个潜在隐患。

### Q7 公共字段自动填充 AOP

**问题**：公共字段自动填充的 AOP 切点表达式怎么写的？INSERT 和 UPDATE 各填哪些字段？`BaseContext.getCurrentId()` 在这里扮演什么角色？如果某个被拦截的方法第一个参数不是实体对象（比如传的是 DTO 或 List），这段代码会出什么问题？

**你的回答**：

> 切点表达式是指这个切面方法作用的范围，具体写的方法我忘记了，但我知道大概是"哪个模块.哪个包.哪个类.哪个方法"这种形式，然后用 * 来代指所有，用 (..) 代指所有方法。INSERT 填充更新时间和更新员工 id、创建时间和创建员工 id；UPDATE 只有更新时间和员工 id。获取当前登录的员工 id，就会将值赋在错误的对象上面，而真正需要赋值的实体对象没有被赋值，甚至因为拦截到的对象中没有相应字段而抛出异常。

**点评 / 标准答案**：

- 真实切点表达式：

```java

@Pointcut("execution(* com.sky.mapper.*.*(..)) && @annotation(com.sky.annotation.AutoFill)")

```

- `execution(...)` 匹配方法执行；`* com.sky.mapper.*.*(..)` = 返回类型任意 + mapper 包下所有类所有方法 + 任意参数；`&& @annotation(AutoFill)` = **方法上必须带 @AutoFill 注解**。
- 注解标在 **mapper 接口方法**上，因为 mapper 方法的第一个参数正好是实体对象。
- INSERT 填 create_time / create_user / update_time / update_user；UPDATE 只填 update_time / update_user；操作者 id 来自 `BaseContext.getCurrentId()`（ThreadLocal）。
- 第一个参数不是实体：`getDeclaredMethod` 反射找不到 setter 会抛 `NoSuchMethodException`（或 NPE），最终包成 `RuntimeException`——所以 `@AutoFill` 方法必须约定第一个参数是带这些字段的实体。
- 细节：`args` 为 null 或长度为 0 时直接 return（安全返回）。

---

## 四、第三层：进阶难点

### Q8 支付与 WebSocket

**问题**：用户支付成功后（paySuccess 方法）做了哪几件事？"来单提醒"和"客户催单"通过 WebSocket 推给管理端的消息，type 字段分别是什么？这两个消息分别由谁触发、在哪个方法里触发的？服务端的 WebSocket 是"一个客户端一条连接"还是"所有人共用一条连接"（看 WebSocketServer 里存的 session 是单个还是集合）？

**你的回答**：

> 将订单状态转换为已支付，更新支付时间、支付方法，并通过 WebSocket 向商家端发送来单提醒。type 是 1 和 2，由 paySuccess 方法和用户催单接口中的催单方法触发。都公用一个连接吧，因为只建立了一个 WebSocket。

**你的追问反驳**：

> 但是我这里只有一个商户，所以只有一条连接耶。

**点评 / 标准答案**：

- `paySuccess` 具体动作：`status → 2 待接单`、`payStatus → 已支付`、`checkoutTime → now`，然后组装 `{type:1, orderId, content:订单号}` 通过 WebSocket 广播。注意要区分"订单状态"和"支付状态"两个字段。
- 触发链路：微信支付平台回调 `/notify/paySuccess` → 读取请求体 → APIv3 密钥 AES 解密 `resource.ciphertext` → 取 `out_trade_no` → 调 `paySuccess` 改状态 + 广播 → 给微信回 `{code: SUCCESS}`。
- 催单：用户端催单接口 → `reminder` 方法 → `type = 2`。
- **连接模型（重点，你的理解需要纠正）**：服务端只有一个端点 `@ServerEndpoint("/ws/{sid}")`，但**每个客户端各有一条独立连接**——`sessionMap` 存 N 条 `sid → Session`；推送是 `sendToAllClient` 遍历所有 Session **全量广播**。"一个商户"不等于"一条连接"：同一个后台开两个标签页就是两条连接，服务端只按连接数存储，不区分商户。
- 加分点：`sessionMap` 用普通 `HashMap` 在多线程下不安全，生产应换 `ConcurrentHashMap`；真实项目一般按门店/商户 id 定向推送或接消息队列。

### Q9 订单超时定时任务

**问题**：订单超时未支付自动取消的定时任务是怎么实现的？OrderTask 用了什么注解、cron 表达式大概是怎样的？它的处理逻辑是什么？这种"定时扫描全表"的方案有什么弊端？如果要改进，你会怎么做？

**你的回答（第一轮）**：

> 使用了 Spring Task 提供的注解 @Scheduled，cron 表达式一般 6~7 个字符，这里超时订单是每分钟执行一次，也就是（1, * , * , * , * , *）。

**你的回答（追问逻辑与弊端后）**：

> 它是每分钟查找所有处于未支付的订单，然后判断每个订单下单时间和当前相比是否已经超过 15 分钟，如果超过 15 分钟就将该订单状态设置为已取消。弊端就是需要将所有处于未支付的订单全部扫描一遍，如果订单量太大，处理起来就太慢了。如果要改进，我目前应该没有什么很好的方法，但是 Redis 给了我一个启发：可以给订单设置一个自动销毁的任务，等 15 分钟一过直接变成已取消？

**点评 / 标准答案**：

- 注解与 cron：`@Scheduled(cron = "0 * * * * ?")`——**6 个字段**（秒 分 时 日 月 周），秒=0、分=*，即每分钟第 0 秒执行；`?` 表示不指定周几（日和周不能同时为 `*`）。你说的"6~7 个字符"不准确，是"6 个字段（可加第 7 个年）"。
- 处理逻辑：`LocalDateTime.now().plusMinutes(-15)` 作为时间阈值，SQL `where status = 1 and order_time < #{orderTime}` 直接查出超时订单（**判断下推到了 SQL**，不是逐条在 Java 里比），循环里补 `status=6、cancel_reason、cancel_time` 后逐条 `update`。
- 兄弟任务：`processDeliveryOrder`，`0 0 1 * * ?` 每天凌晨 1 点，把派送中超过 1 小时的订单置为已完成。
- **四个弊端**：
  1. 全表扫描未支付订单，量大时慢；
  2. **逐条 update**，N 条订单 N 次数据库往返（可一条 `UPDATE ... WHERE status=1 AND order_time < ...` 批量完成）；
  3. 最多 1 分钟延迟，不是实时；
  4. **集群部署会重复执行**，需要分布式锁（ShedLock / Redis 锁）。
- **改进方案**：主方案用 **MQ 延迟消息**（下单发 15 分钟延迟消息，RabbitMQ 死信队列/延迟插件或 RocketMQ 延迟等级，到期消费处理），再留一个**低频定时扫描兜底**防丢。
- 对 Redis 思路的评估：方向对（延迟触发代替轮询），即 Redis 过期键通知方案；但**过期事件不保证及时、不保证不丢**（惰性删除+定期删除机制、重启/内存淘汰会丢），且事件只带 key 不带 value，只能当辅助手段。

### Q10 数据统计与报表

**问题**：营业额统计 sumByMap 的 SQL 思路是什么？为什么统计营业额/订单数时要限定 status = 5（已完成）？销量 Top10 的 SQL 是怎么写的（涉及哪两张表、怎么分组排序）？最后运营数据报表的 Excel 导出大概是什么流程（用了模板还是现写样式，怎么填充数据）？

**你的回答**：

> 通过 sum 聚合函数查询，时间在开始和结束之间，并且订单要已完成，因为只有已完成的订单才是真正的收入，取消的订单不算。Top10 使用了 sum 聚合函数统计所有销售菜品数量，涉及到 order 表和 order_detail 表，按菜品 id 分组，通过销售数量降序排列。首先创建 excel 模板，然后利用 POI 工具创建 excel 对象，调用 excel 中的方法，挨着 sheet 对象、row 对象，以及调用其中的方法一个 cell 一个 cell 填充数据，其中数据来源于调用 mapper 查询出来。

**点评 / 标准答案**：

- **营业额统计**：按天生成日期列表（begin 到 end），每天用 `LocalTime.MIN ~ LocalTime.MAX` 作时间范围，`sum(amount) where status = 5`。只有已完成订单的钱才是真实到账收入；取消/拒单（状态 6）被排除。
- **销量 Top10**：

```sql

select od.name, sum(od.number) number

from order_detail od

left join orders o on od.order_id = o.id

where o.status = 5 and o.order_time > #{beginTime} and o.order_time <= #{endTime}

group by od.name

order by number desc

limit 10

```

- 关键点：**按 `od.name` 分组，不是按菜品 id**——order_detail 存的是下单时的菜品名称快照，没有可靠的外键指向菜品表。
- **Excel 导出**：加载模板（`XSSFWorkbook` 读 classpath 的 `template/运营数据报表模板.xlsx`）→ 取 `sheet1` → 固定坐标填值（第 2 行时间、第 4/5 行汇总指标）→ 循环 30 天从第 8 行写每日明细 → `excel.write(response 输出流)` 下载。**样式由模板预置，POI 只填值**。

### Q11 工作台

**问题**：getBusinessData 里的四个指标分别怎么算的——今日营业额、有效订单数、平均客单价、订单完成率？特别是客单价和完成率，分母分别是什么？另外新增用户数查的是哪张表、按什么时间字段统计？顺带说一下 getOrderOverView 里的待接单、待派送、已完成、已取消分别对应订单状态几？

**你的回答**：

> 今日营业额、订单数直接调用 mapper 使用 sum 聚合函数查询 order 表，统计总的 amount；还有使用 count 聚合函数统计已完成订单总的订单数，要注意的是要使用 beginTime 和 endTime 约束在今天时间查询。平均客单价就用查出来的营业额除以订单数，就是平均客单价。完成率要用完成订单数除以总订单数。新增用户查的是 user 表中的数量，按创建时间是否等于传输进去的时间字段来查询。最后一个问题我觉得问的没有含金量，因为项目中已经用常量来代替了数字订单状态，问这个没有意义。

**点评 / 标准答案**：

- 今日营业额 = `sum(amount)`，`status = 5` + 今天时间范围；
- 有效订单数 = `count(id)`，`status = 5` + 时间范围；
- 平均客单价 = **营业额 ÷ 有效订单数**（分母是有效订单，不是总订单）；
- 订单完成率 = **有效订单数 ÷ 总订单数**（分子已完成，分母当天全部订单）；
- 新增用户：`user` 表按 `create_time` **范围查询**（`>= beginTime and <= endTime`），不是"等于"。
- 状态映射（虽然代码用常量，但业务语义要说得清）：**待接单 = 2、待派送 = 3（不是 4）、派送中 = 4、已完成 = 5、已取消 = 6**；`getOrderOverView` 统计 2 / 3 / 5 / 6 / 全部，不含 4。
- 对"没含金量"的回应：你说项目用常量代替数字、问数字没意义，这个规范意识是对的；但考点不是背数字，而是**状态语义**——尤其"待派送 = 3 而不是 4"这种容易混的点。

---

## 五、技术栈

| 分类   | 技术                                                           |
| ---- | ------------------------------------------------------------ |
| 基础框架 | Spring Boot 2.7.3、Spring MVC、Lombok                          |
| 持久层  | MyBatis + XML 映射、PageHelper 分页、Druid 连接池、MySQL               |
| 缓存   | Spring Data Redis、Spring Cache（`@Cacheable` / `@CacheEvict`） |
| 鉴权   | JWT（JJWT 0.9.1）、双端拦截器、ThreadLocal 上下文                        |
| 接口文档 | Knife4j（Swagger 2.0）                                         |
| 实时通信 | WebSocket（来单提醒、客户催单）                                         |
| 任务调度 | Spring `@Scheduled` 定时任务                                     |
| 文件服务 | 阿里云 OSS                                                      |
| 支付   | 微信支付 V3 API（wechatpay-apache-httpclient）                     |
| 报表导出 | Apache POI（XSSFWorkbook + 模板填充）                              |
| 其他   | AOP 切面、统一异常处理、fastjson                                       |
