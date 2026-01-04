# Effective On-Hardware Fuzzing of Embedded Operating Systems

<table>
  <tr>
    <td>
      <b>
	  📌 Title: Effective On-Hardware Fuzzing of Embedded Operating Systems
	</b>
    </td>
  </tr>
  <tr>
    <td>
      <b>👨‍🎓 Author: </b>
	  Yuheng Shen (Tsinghua University), Jianzhong Liu (Shandong University), Qiming Guo (Beihang University), Yifei Chu (Tsinghua University), Qiang Zhang (Hunan University), Heyuan Shi (Central South University), Yu Jiang (Tsinghua University) 
    </td>
  </tr>
  <tr>
    <td>
      <b>🏛️ Publication: </b>
	  EUROSYS 26
   </td>
  </tr>
  <tr>
    <td>
      <b>📜 Journal Tags:  </b>
	  fault injection, distributed systems, configuration testing, fuzzing, fault tolerance, CAFault, FDModel, bug detection
    </td>
  </tr>
  <tr>
    <td>
      <b>🎯 Tags:   </b>
	  无
    </td>
  </tr>
  <tr>
    <td>
      <b>🔗 DOI:   </b>
	  <a href="10.1145/3767295.3769326">10.1145/3767295.3769326</a>
    </td>
  </tr>
  <tr>
    <td>
      <b>📁 Local Link:   </b>
	  无
    </td>
  </tr>
  <tr>
    <td>
      <b>📅 Note Date:   </b>
	  2025/10/10
    </td>
  </tr>

---

  <h2 style="color: #e0ffff; background-color: #66cdaa">📜 研究背景 现状 目标</h2>
  <hr/>

### ⚙️ 背景

- **嵌入式操作系统**：是为数十亿物联网设备提供动力的关键，其安全性和稳定性至关重要

  - 但由于其**多样性**（硬件依赖强、API 不统一）和**资源约束**（内存小、无 MMU），导致极易出现安全漏洞
  - 所以需要高效的测试工具——模糊测试来提前发现 bug

  

- **模糊测试**：是软件测试的核心技术，通过随机生成测试用例，监控系统是否异常，从而发现 bug

  - 但现有的嵌入式操作系统模糊技术仍无法在硬件上进行全系统测试，原因如下：
    1. 模拟器依赖，无法覆盖真实硬件
    2. 应用层测试，测不到内核深层逻辑



### 💡 现状

- 


### 🚀 目标

- 设计并实现一款**基于真实硬件的反馈引导式模糊测试工具（EOF）**

---

  <h2 style="color: #20b2aa; background-color: #afeeee">🔁 研究内容</h2>
  <hr />

### 🚊 研究基础

- **分布式系统的故障**：网络延迟、数据包丢失、硬件故障

  

- **分布式系统的容错机制**：
  - 复制：在多个不同的节点上创建副本
  - 共识协议：多个节点通过互相通信和投票的方式，对某个决策达成一致意见，如 Raft 共识协议
  - 故障转移策略：系统通过故障检测机制发现某个节点故障后，自动将该节点的工作任务转移到备用节点上

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);
    width: 66%; height: auto;" 
    src="figs/FT_DistributedSystems_Fig1.jpg">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">分布式系统中的典型容错机制，受各种配置设置的影响</div>
</center>

### 💻 技术路线

- **CAFault**：一个配置感知的故障注入框架

  1. 模块一：**FDModel（故障 - 配置依赖模型）—— 剪枝配置空间**

  2. 模块二：**故障处理引导的模糊测试 —— 优化故障空间探索**

     <center>
         <img style="border-radius: 0.3125em;
         box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);
         width: 60%; height: auto;" 
         src="figs/CAFault_Workflow_Fig1.jpg">
         <br>
         <div style="color:orange; border-bottom: 1px solid #d9d9d9;
         display: inline-block;
         color: #999;
         padding: 2px;">CAFault 的工作流程，包括隐式依赖构造和故障处理引导模糊化两个主要阶段</div>
     </center>



- **（模块一）FDModel 的更新**：动态学习 “配置项 - 故障” 的隐式依赖，只保留对容错逻辑有影响的配置，减少无效探索

  - **核心思路**：同一故障下，若修改某个配置导致容错逻辑的覆盖率变化，则两者存在依赖。将测试分布式系统过程形式化定义为：
    $$
    φ = \{ C，F，Cov \}
    $$

  - **FDModel 构建步骤**：
    1. 初始化配置集：从系统默认配置开始，放入待探索配置集 $C$
    2. 配置变异：从 $C$ 中取出一个配置 $c$ ，随机变异部分配置项（如布尔型取反、数值型加减），生成新配置 $c'$
    3. 覆盖率对比：
       - 加载配置 $c$，注入所有故障（如节点崩溃、网络延迟），记录容错逻辑的覆盖率 $Cov$
       - 加载配置 $c'$ ，重复注入相同故障，记录覆盖率 $Cov'$
       - 若 $Cov≠Cov'$ ，说明变异的配置项与这些故障存在依赖
    4. 依赖最小化：用二分法筛选出最小影响配置集（eg：若变异 3 个配置项导致覆盖率变化，二分法逐步排除无影响项，最终定位到 1 个关键配置）
    5. 更新 FDModel：将 “配置项 - 故障” 的依赖关系存入模型，同时将 $c'$（高价值配置）加入 $C$ ，用于后续测试



- **（模块二）故障处理引导的模糊测试**



### 👩🏻‍💻 研究方法 方案



### 🧩 数据



### 🔬 实验



### 📜 结论

- **CAFault** 在 HDFS、ZooKeeper、MySQL-Cluster 等分布式系统的容错逻辑覆盖率比 CrashFuzz、Mallory 和 Chronos 分别提高了 31.5 %、29.3 %和81.5 %，并检测出了 **16** 个严重的未知错误

  

- **3 个贡献**：
  
  1. 提出 FDModel 
  2. 引入故障处理引导模糊测试
  3. 在 4 个主流分布式系统上进行工程验证

### 📚 参考



---

  <h2 style="color: #004d99; background-color: #87cefa">🤔 个人总结</h2>
  <hr />

### 🙋‍♀️ 重点记录

- 
  

### 📌 待解决



### 💭 思考启发
