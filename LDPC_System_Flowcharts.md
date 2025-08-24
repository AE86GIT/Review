# LDPC系统完整流程图分析

## 1. 系统整体启动流程图

```mermaid
flowchart TD
    A[程序启动] --> B{检查运行模式}
    B -->|SIMP_TST模式| C[mainTsk函数]
    B -->|完整系统模式| D[stTsk函数]
    
    C --> E[系统初始化]
    D --> E
    
    E --> F[设置产品信息]
    F --> G[sysInitialize系统初始化]
    G --> H[读取配置文件]
    H --> I[初始化LDPC参数]
    I --> J[创建数据库连接]
    J --> K[创建子任务]
    K --> L[启动仿真循环]
    
    L --> M{运行模式}
    M -->|守护进程模式| N[while1 pause循环]
    M -->|交互模式| O[sysCliLogin交互界面]
    
    N --> P[定期状态检查]
    O --> P
    P --> Q[输出统计结果]
    Q --> R[程序结束]
    
    style A fill:#e1f5fe
    style R fill:#c8e6c9
    style M fill:#fff3e0
```

## 2. 主任务创建和调度流程图

```mermaid
flowchart TD
    A[mainTsk开始] --> B[读取任务配置]
    B --> C[初始化LDPC参数]
    C --> D[创建互斥锁]
    D --> E[打开数据库]
    E --> F[确定子任务数量]
    F --> G[设置任务平衡模式]
    
    G --> H{任务平衡模式}
    H -->|TBM_BLK| I[按错误块数分配]
    H -->|TBM_SNB| J[按SNB值分配]
    H -->|TBM_MTX| K[按输入矩阵分配]
    
    I --> L[创建子任务循环]
    J --> L
    K --> L
    
    L --> M[分配任务参数]
    M --> N[创建线程]
    N --> O[启动subTskPrc函数]
    O --> P[任务计数器递增]
    P --> Q{是否创建完所有任务}
    Q -->|否| L
    Q -->|是| R[进入监控循环]
    
    R --> S[定期调用ldpcStsCheck]
    S --> T[输出统计信息]
    T --> R
    
    style A fill:#e1f5fe
    style R fill:#fff3e0
```

## 3. 子任务处理流程图

```mermaid
flowchart TD
    A[subTskPrc开始] --> B[获取任务ID]
    B --> C[初始化LDPC参数]
    C --> D[分配内存空间]
    D --> E[初始化矩阵]
    E --> F[设置任务状态为ACTIVE]
    
    F --> G[主仿真循环开始]
    G --> H[初始化统计参数]
    H --> I[计算SNR值]
    I --> J[计算噪声方差]
    J --> K[设置随机种子]
    
    K --> L[仿真帧循环]
    L --> M[生成随机信息比特]
    M --> N[LDPC编码]
    N --> O[BPSK调制]
    O --> P[AWGN信道传输]
    
    P --> Q[计算LLR]
    Q --> R[LDPC解码初始化]
    R --> S[迭代解码循环]
    
    S --> T{解码算法选择}
    T -->|Flooding BP| U[Flooding_BPLx]
    T -->|Layered BP| V[C_SBP]
    T -->|EDSRBP| W[EDSRBP]
    T -->|CNR Schedule| X[CNR_LBP]
    T -->|Pin Flooding| Y[Pin_Flooding_BP]
    T -->|RBP| Z[RBP]
    
    U --> AA[停止准则检查]
    V --> AA
    W --> AA
    X --> AA
    Y --> AA
    Z --> AA
    
    AA --> BB{停止条件}
    BB -->|未满足| S
    BB -->|满足| CC[提取解码结果]
    
    CC --> DD[计算比特错误]
    DD --> EE[更新统计信息]
    EE --> FF{是否达到最大错误块数}
    FF -->|否| L
    FF -->|是| GG[输出仿真结果]
    
    GG --> HH[插入数据库记录]
    HH --> II[设置任务状态为IDLE]
    II --> JJ[子任务结束]
    
    style A fill:#e1f5fe
    style JJ fill:#c8e6c9
    style S fill:#fff3e0
```

## 4. 系统状态机流程图

```mermaid
stateDiagram-v2
    [*] --> TS_NONE: 系统启动
    
    TS_NONE --> TS_IDLE: 任务创建完成
    TS_IDLE --> TS_ACTIVE: 开始仿真
    TS_ACTIVE --> TS_IDLE: 仿真完成
    
    TS_ACTIVE --> TS_ACTIVE: 继续仿真循环
    
    TS_IDLE --> TS_IDLE: 等待新任务
    TS_IDLE --> [*]: 系统关闭
    
    note right of TS_NONE
        初始状态
        任务未分配
    end note
    
    note right of TS_IDLE
        空闲状态
        任务已创建但未运行
        等待调度
    end note
    
    note right of TS_ACTIVE
        活跃状态
        正在执行LDPC仿真
        包含编码、传输、解码
    end note
```

## 5. 任务平衡模式状态机

```mermaid
stateDiagram-v2
    [*] --> TBM_BLK: 按错误块数平衡
    
    TBM_BLK --> TBM_SNB: 切换到SNB模式
    TBM_SNB --> TBM_MTX: 切换到矩阵模式
    TBM_MTX --> TBM_BLK: 切换到块模式
    
    note right of TBM_BLK
        按错误块数分配任务
        MaxErrBlk/tskNum
        适合固定SNR仿真
    end note
    
    note right of TBM_SNB
        按SNB值分配任务
        每个任务处理不同SNR
        适合SNR扫描仿真
    end note
    
    note right of TBM_MTX
        按输入矩阵分配任务
        每个任务处理不同矩阵
        适合多矩阵对比
    end note
```

## 6. 多线程加速原理流程图

```mermaid
flowchart TD
    A[主线程] --> B[创建多个子线程]
    B --> C[线程0: subTskPrc]
    B --> D[线程1: subTskPrc]
    B --> E[线程2: subTskPrc]
    B --> F[线程N: subTskPrc]
    
    C --> G[独立内存空间]
    D --> H[独立内存空间]
    E --> I[独立内存空间]
    F --> J[独立内存空间]
    
    G --> K[并行LDPC仿真]
    H --> L[并行LDPC仿真]
    I --> M[并行LDPC仿真]
    J --> N[并行LDPC仿真]
    
    K --> O[共享数据库]
    L --> O
    M --> O
    N --> O
    
    O --> P[互斥锁保护]
    P --> Q[统计结果汇总]
    Q --> R[性能提升]
    
    style A fill:#e1f5fe
    style R fill:#c8e6c9
    style P fill:#fff3e0
```

## 7. 线程间通信和同步机制

```mermaid
flowchart TD
    A[主线程] --> B[创建互斥锁]
    B --> C[cdrLock: 数据库锁]
    B --> D[stsLock: 统计锁]
    
    C --> E[子线程0]
    C --> F[子线程1]
    C --> G[子线程N]
    
    E --> H[数据库写入]
    F --> H
    G --> H
    
    H --> I[互斥访问]
    I --> J[原子操作]
    
    D --> K[统计信息更新]
    K --> L[线程安全统计]
    L --> M[实时性能监控]
    
    style A fill:#e1f5fe
    style I fill:#fff3e0
    style M fill:#c8e6c9
```

## 8. LDPC解码算法选择流程图

```mermaid
flowchart TD
    A[解码开始] --> B{解码模式选择}
    
    B -->|LDPCDecMode=0| C[Flooding BP]
    B -->|LDPCDecMode=1| D[Layered BP]
    B -->|LDPCDecMode=2| E[EDSRBP]
    B -->|LDPCDecMode=3| F[CNR Schedule]
    B -->|LDPCDecMode=4| G[Pin Flooding BP]
    B -->|LDPCDecMode=5| H[RBP]
    
    C --> I[Flooding_BPLx函数]
    D --> J[C_SBP函数]
    E --> K[EDSRBP函数]
    F --> L[CNR_LBP函数]
    G --> M[Pin_Flooding_BP函数]
    H --> N[RBP函数]
    
    I --> O[停止准则检查]
    J --> O
    K --> O
    L --> O
    M --> O
    N --> O
    
    O --> P{停止条件}
    P -->|未满足| Q[继续迭代]
    P -->|满足| R[解码完成]
    
    Q --> B
    
    style A fill:#e1f5fe
    style R fill:#c8e6c9
    style O fill:#fff3e0
```

## 9. 停止准则流程图

```mermaid
flowchart TD
    A[停止准则检查] --> B{准则类型}
    
    B -->|StopCriteria=0| C[标准停止准则]
    B -->|StopCriteria=1| D[CSA准则]
    B -->|StopCriteria=2| E[CMM准则]
    B -->|StopCriteria=3| F[CAET准则]
    B -->|StopCriteria=4| G[JPAN准则]
    
    C --> H[Hn_check检查]
    D --> I[CSA_Criterion]
    E --> J[CMM_Criterion]
    F --> K[CAET_Criterion]
    G --> L[JPAN_Criterion]
    
    H --> M{检查结果}
    I --> M
    J --> M
    K --> M
    L --> M
    
    M -->|c==0 或 iter==MaxIter| N[停止解码]
    M -->|其他情况| O[继续迭代]
    
    N --> P[返回停止标志]
    O --> Q[返回继续标志]
    
    style A fill:#e1f5fe
    style P fill:#c8e6c9
    style Q fill:#fff3e0
```

## 10. 数据库操作流程图

```mermaid
flowchart TD
    A[数据库操作] --> B{操作类型}
    
    B -->|创建数据库| C[openCdrDb]
    B -->|插入记录| D[insert_cdrTbl]
    B -->|插入中间记录| E[insert_midcdrTbl]
    B -->|关闭数据库| F[closeCdrDb]
    
    C --> G[检查数据库文件]
    G --> H{文件是否存在}
    H -->|存在| I[打开现有数据库]
    H -->|不存在| J[创建新数据库]
    
    J --> K[创建表结构]
    K --> L[cdr_tbl表]
    K --> M[mid_cdr_tbl表]
    
    I --> N[数据库就绪]
    L --> N
    M --> N
    
    D --> O[获取互斥锁]
    E --> O
    O --> P[执行SQL语句]
    P --> Q[释放互斥锁]
    Q --> R[操作完成]
    
    F --> S[关闭连接]
    S --> T[释放资源]
    
    style A fill:#e1f5fe
    style R fill:#c8e6c9
    style T fill:#c8e6c9
    style O fill:#fff3e0
```

## 11. 性能监控和统计流程图

```mermaid
flowchart TD
    A[性能监控开始] --> B[ldpcStsCheck函数]
    B --> C[遍历所有子任务]
    C --> D[获取任务统计信息]
    D --> E[计算性能指标]
    
    E --> F[比特错误率]
    E --> G[帧错误率]
    E --> H[平均迭代次数]
    E --> I[每秒迭代次数]
    
    F --> J[输出统计表格]
    G --> J
    H --> J
    I --> J
    
    J --> K[插入数据库记录]
    K --> L[计算总体性能]
    L --> M[输出汇总信息]
    M --> N[更新监控状态]
    N --> O[等待下次检查]
    
    O --> P{是否继续监控}
    P -->|是| B
    P -->|否| Q[监控结束]
    
    style A fill:#e1f5fe
    style Q fill:#c8e6c9
    style J fill:#fff3e0
```

## 12. 内存管理流程图

```mermaid
flowchart TD
    A[内存分配] --> B[stskPmdStsAlloc函数]
    B --> C[分配统计内存]
    C --> D[TotalBitErr数组]
    C --> E[TotalFrameErr数组]
    
    B --> F[分配数据内存]
    F --> G[source_data]
    F --> H[encoded_data]
    F --> I[decoded_data]
    F --> J[received_data]
    
    B --> K[分配LLR内存]
    K --> L[lratio数组]
    K --> M[LLR数组]
    K --> N[LLR_Inital数组]
    
    B --> O[分配算法内存]
    O --> P[lamita数组]
    O --> Q[lao数组]
    O --> R[PinPos数组]
    
    D --> S[内存初始化]
    E --> S
    G --> S
    H --> S
    I --> S
    J --> S
    L --> S
    M --> S
    N --> S
    P --> S
    Q --> S
    R --> S
    
    S --> T[内存就绪]
    T --> U[开始仿真]
    
    style A fill:#e1f5fe
    style T fill:#c8e6c9
    style S fill:#fff3e0
```

## 总结

这个LDPC系统采用了以下关键技术：

1. **多线程并行处理**：通过创建多个子线程同时进行LDPC仿真，显著提升计算性能
2. **任务平衡机制**：支持多种任务分配策略，适应不同的仿真需求
3. **状态机管理**：清晰的任务状态转换，确保系统稳定运行
4. **互斥同步**：通过互斥锁保护共享资源，保证数据一致性
5. **实时监控**：定期检查各线程状态，输出性能统计信息
6. **数据库记录**：将仿真结果持久化存储，便于后续分析

整个系统设计合理，具有良好的可扩展性和稳定性，能够高效地进行LDPC码的性能仿真。 