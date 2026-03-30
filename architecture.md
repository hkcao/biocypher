# BioCypher + BioChatter 技术架构图

## 一、总体架构全景图

```mermaid
graph TB
    subgraph UserLayer["👤 用户层"]
        UQ["自然语言问题<br/>'What proteins interact with p53?'"]
        UA["数据适配器 (用户编写)<br/>Python Generator 产出元组"]
        SC["schema_config.yaml<br/>定义知识图谱 Schema"]
        BC["biocypher_config.yaml<br/>全局配置 (DBMS/模式等)"]
    end

    subgraph BioChatter["🔍 BioChatter — 查询层"]
        direction TB
        CQ["Conversation.query()<br/>conversation.py:565<br/>━━ 唯一用户入口 ━━"]
        IC["_inject_context()<br/>conversation.py:1127<br/>RAG 上下文注入"]

        subgraph AgentDispatch["Agent 分发机制"]
            direction LR
            RAS["RagAgentSelector<br/>selector_agent.py<br/>LLM 智能路由(可选)"]
            RA["RagAgent<br/>rag_agent.py<br/>统一封装层"]
        end

        subgraph RetrievalBackends["检索后端"]
            direction TB
            DA["DatabaseAgent<br/>database_agent.py<br/>KG 查询执行器"]
            VA["VectorDatabaseAgent<br/>vectorstore_agent.py<br/>Milvus 向量搜索"]
            AA["APIAgent<br/>api_agent/<br/>BLAST / OncoKB"]
        end

        subgraph QueryGeneration["Cypher 查询生成 (Schema感知)"]
            direction TB
            PE["BioCypherPromptEngine<br/>prompts.py:11"]
            SE["Step1: _select_entities()<br/>LLM 选择实体类型"]
            SR["Step2: _select_relationships()<br/>LLM 选择关系类型"]
            SP["Step3: _select_properties()<br/>LLM 选择属性"]
            GQ["Step4: _generate_query()<br/>LLM 生成 Cypher"]
            KR["KGQueryReflexionAgent<br/>kg_langgraph_agent.py<br/>LangGraph 迭代优化(可选)"]
        end

        PQ["_primary_query()<br/>LLM 综合上下文生成回答"]
    end

    subgraph BioCypher["🏗️ BioCypher — 知识图谱构建层"]
        direction TB
        CORE["BioCypher()<br/>_core.py:42<br/>━━ 主编排器 ━━"]

        subgraph OntologySystem["本体系统 (Ontology System)"]
            direction TB
            OM["OntologyMapping<br/>_mapping.py<br/>Schema 配置解析"]
            ONT["Ontology<br/>_ontology.py<br/>本体混合管理器"]
            OA_H["OntologyAdapter (Head)<br/>加载主本体 (Biolink)"]
            OA_T["OntologyAdapter (Tail)<br/>加载扩展本体 (Mondo等)"]
        end

        subgraph TranslationPipeline["翻译管线"]
            direction TB
            TR["Translator<br/>_translate.py<br/>数据翻译 + 映射"]
            CR["BioCypherNode / Edge<br/>_create.py<br/>标准化数据对象"]
            DD["Deduplicator<br/>_deduplicate.py<br/>全局去重"]
        end

        subgraph OutputAdapters["输出适配器"]
            direction TB
            subgraph OfflineWrite["离线写入 (output/write/)"]
                NW["Neo4jBatchWriter<br/>→ CSV + import脚本"]
                PW["PostgreSQLWriter<br/>→ SQL文件"]
                SW["SQLiteWriter"]
                RW["RDFWriter → Turtle"]
                OW["OWLWriter"]
            end
            subgraph OnlineConnect["在线连接 (output/connect/)"]
                ND["Neo4jDriver<br/>→ 实时写入Neo4j"]
            end
            subgraph InMemory["内存模式 (output/in_memory/)"]
                NX["NetworkxKG"]
                PD["PandasKG"]
            end
        end
    end

    subgraph ExternalSystems["外部系统"]
        NEO4J[("Neo4j<br/>图数据库")]
        PGSQL[("PostgreSQL")]
        MILVUS[("Milvus<br/>向量数据库")]
        OWL_FILE["OWL/RDF 本体文件<br/>(Biolink, Mondo, GO...)"]
        LLM_API["LLM API<br/>(OpenAI/Claude/Gemini)"]
    end

    %% 用户层 → BioCypher
    UA -->|"(id, type, props) 元组"| CORE
    SC -->|"schema 定义"| OM
    BC -->|"全局配置"| CORE
    OWL_FILE -->|"本体文件"| OA_H
    OWL_FILE -->|"本体文件"| OA_T

    %% BioCypher 内部流程
    CORE --> OM
    OM --> ONT
    ONT --> OA_H
    ONT --> OA_T
    CORE --> TR
    TR -->|"查本体映射"| ONT
    TR --> CR
    CR --> DD
    DD --> NW
    DD --> PW
    DD --> SW
    DD --> RW
    DD --> ND
    DD --> NX
    NW -->|"CSV导入"| NEO4J
    ND -->|"Bolt协议"| NEO4J
    PW --> PGSQL

    %% BioCypher → BioChatter 桥接
    CORE -->|"write_schema_info()<br/>导出 schema_info"| PE

    %% 用户层 → BioChatter
    UQ --> CQ
    CQ --> IC
    IC -->|"selector模式"| RAS
    IC -->|"遍历模式"| RA
    RAS -->|"选择最佳agent"| RA
    RA -->|"mode=kg"| DA
    RA -->|"mode=vectorstore"| VA
    RA -->|"mode=api_*"| AA

    %% KG查询路径
    DA --> PE
    PE --> SE --> SR --> SP --> GQ
    DA -->|"reflexion模式"| KR
    KR -->|"迭代优化"| GQ
    GQ -->|"Cypher查询"| NEO4J
    DA -->|"查询结果"| IC

    %% 其他检索路径
    VA -->|"向量搜索"| MILVUS
    VA -->|"搜索结果"| IC

    %% LLM 调用
    SE -.->|"LLM调用"| LLM_API
    SR -.->|"LLM调用"| LLM_API
    SP -.->|"LLM调用"| LLM_API
    GQ -.->|"LLM调用"| LLM_API
    KR -.->|"LLM调用"| LLM_API
    RAS -.->|"LLM调用"| LLM_API

    %% RAG注入 → 最终回答
    IC -->|"检索结果注入<br/>system message"| PQ
    PQ -.->|"LLM生成回答"| LLM_API

    %% 样式
    classDef core fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef ontology fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef translate fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px
    classDef query fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef external fill:#fce4ec,stroke:#b71c1c,stroke-width:2px
    classDef entry fill:#ffeb3b,stroke:#f57f17,stroke-width:3px

    class CORE,CQ entry
    class OM,ONT,OA_H,OA_T ontology
    class TR,CR,DD translate
    class PE,SE,SR,SP,GQ,KR,DA query
    class NEO4J,PGSQL,MILVUS,LLM_API,OWL_FILE external
```

## 二、本体系统详细架构图

```mermaid
graph TB
    subgraph Input["输入源"]
        OWL["OWL/RDF/TTL 本体文件"]
        YAML["schema_config.yaml"]
    end

    subgraph OntologyAdapter["OntologyAdapter (_ontology.py:30)"]
        direction TB
        PARSE["RDFLib.Graph.parse()<br/>解析本体文件"]
        CONVERT["rdflib_to_networkx_digraph()<br/>RDF → NetworkX DiGraph"]
        PROCESS["后处理"]

        subgraph ProcessDetails["后处理步骤"]
            direction LR
            P1["解析 rdfs:subClassOf<br/>构建继承链"]
            P2["处理 owl:intersectionOf<br/>多重继承"]
            P3["去除 URI 前缀<br/>http://... → label"]
            P4["标签标准化<br/>→ lower sentence case"]
            P5["祖先剪枝<br/>只保留 root 可达节点"]
        end
    end

    subgraph OntologyMapping_["OntologyMapping (_mapping.py)"]
        direction TB
        READ_YAML["读取 schema_config.yaml"]
        BUILD_SCHEMA["构建 extended_schema"]

        subgraph InheritanceLogic["继承机制"]
            direction LR
            VI["纵向继承<br/>子实体继承父属性"]
            HI["横向继承<br/>多preferred_id→虚拟叶"]
            PI["属性继承<br/>inherit_properties: true"]
        end

        ES["extended_schema 字典<br/>┌─────────────────────────┐<br/>│ protein:                │<br/>│   represented_as: node  │<br/>│   preferred_id: uniprot │<br/>│   input_label: protein  │<br/>│   properties:           │<br/>│     name: str           │<br/>│     score: float        │<br/>│   is_a: polypeptide     │<br/>└─────────────────────────┘"]
    end

    subgraph Ontology_["Ontology (_ontology.py) — 混合本体管理器"]
        direction TB
        LOAD_HEAD["① 加载 Head Ontology<br/>(如 Biolink Model)"]
        LOAD_TAIL["② 加载 Tail Ontologies<br/>(如 Mondo, GO, SO)"]
        JOIN["③ _join_ontologies()<br/>将 tail 嫁接到 head 的指定节点"]
        EXTEND["④ _extend_ontology()<br/>将 schema 自定义实体加入本体树"]
        ADD_PROPS["⑤ 添加 properties<br/>从 schema 注入属性定义"]
        TREE["最终: 完整的混合本体 DiGraph"]
    end

    subgraph HybridExample["混合本体示例"]
        direction TB
        ROOT["entity (root)"]
        BIO_ENT["biological entity"]
        GENE["gene"]
        PROTEIN["protein"]
        DISEASE["disease ← (嫁接点)"]
        CANCER["cancer (来自 Mondo)"]
        DIABETES["diabetes (来自 Mondo)"]
        CUSTOM["my_custom_type (来自 schema)"]

        ROOT --> BIO_ENT
        BIO_ENT --> GENE
        BIO_ENT --> PROTEIN
        BIO_ENT --> DISEASE
        DISEASE -->|"tail join"| CANCER
        DISEASE -->|"tail join"| DIABETES
        BIO_ENT -->|"schema extend"| CUSTOM
    end

    subgraph Downstream["下游消费者"]
        TRANSLATOR["Translator<br/>用 input_label 查映射<br/>→ 确定 node_label"]
        PROMPT_ENG["BioCypherPromptEngine<br/>(BioChatter)<br/>用 schema 约束 LLM 查询生成"]
    end

    %% 连接
    OWL --> PARSE --> CONVERT --> PROCESS
    PROCESS --> P1 & P2 & P3 & P4 & P5
    YAML --> READ_YAML --> BUILD_SCHEMA
    BUILD_SCHEMA --> VI & HI & PI
    VI & HI & PI --> ES

    P5 -->|"Head DiGraph"| LOAD_HEAD
    P5 -->|"Tail DiGraph"| LOAD_TAIL
    ES -->|"mapping"| EXTEND

    LOAD_HEAD --> JOIN
    LOAD_TAIL --> JOIN
    JOIN --> EXTEND --> ADD_PROPS --> TREE

    TREE --> TRANSLATOR
    TREE --> PROMPT_ENG
    ES --> TRANSLATOR
    ES --> PROMPT_ENG

    %% 样式
    classDef ontCore fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef schema fill:#e8eaf6,stroke:#1a237e,stroke-width:2px
    classDef example fill:#f1f8e9,stroke:#33691e,stroke-width:1px,stroke-dasharray: 5 5
    classDef downstream fill:#fff3e0,stroke:#e65100,stroke-width:2px

    class PARSE,CONVERT,PROCESS,P1,P2,P3,P4,P5,LOAD_HEAD,LOAD_TAIL,JOIN,EXTEND,ADD_PROPS,TREE ontCore
    class READ_YAML,BUILD_SCHEMA,VI,HI,PI,ES schema
    class ROOT,BIO_ENT,GENE,PROTEIN,DISEASE,CANCER,DIABETES,CUSTOM example
    class TRANSLATOR,PROMPT_ENG downstream
```

## 三、数据入库管线详细图

```mermaid
graph LR
    subgraph UserAdapter["用户适配器"]
        NG["node_generator()<br/>yield (id, type, props)"]
        EG["edge_generator()<br/>yield (id, src, tar, type, props)"]
    end

    subgraph Translate["Translator (_translate.py)"]
        direction TB
        TE["translate_entities()"]

        subgraph NodePath["节点翻译 (L70-138)"]
            N1["解包: _id, _type, _props"]
            N2["_get_ontology_mapping(_type)<br/>查 input_label → 本体类"]
            N3{{"找到映射?"}}
            N4["_filter_props()<br/>白名单/黑名单过滤"]
            N5["yield BioCypherNode"]
            N6["_record_no_type()<br/>⚠️ 静默丢弃 + 计数"]
        end

        subgraph EdgePath["边翻译 (L198-319)"]
            E1["解包: _id, _src, _tar, _type, _props"]
            E2["_get_ontology_mapping(_type)"]
            E3{{"找到映射?"}}
            E4["_filter_props()"]
            E5{{"represented_as?"}}
            E6["yield BioCypherEdge"]
            E7["yield BioCypherRelAsNode<br/>(node + 2条边)"]
            E8["_record_no_type()<br/>⚠️ 静默丢弃"]
        end

        subgraph FilterLogic["_filter_props() 逻辑 (L152-196)"]
            F1["schema 定义了白名单?"]
            F2["保留白名单内属性<br/>多余属性丢弃"]
            F3["缺失属性补 None"]
            F4["排除 exclude_properties"]
            F5["无白名单 → 全部保留"]
        end
    end

    subgraph StrictMode["严格模式检查"]
        SM{{"strict_mode?"}}
        SM_Y["必须包含:<br/>source, licence, version<br/>缺失 → raise ValueError"]
        SM_N["跳过检查"]
    end

    subgraph Dedup["Deduplicator"]
        D1{{"已见过?"}}
        D2["✅ 通过"]
        D3["❌ 跳过 (去重)"]
    end

    subgraph Writers["Writer 写入"]
        W1["Neo4jBatchWriter → CSV"]
        W2["PostgreSQLWriter → SQL"]
        W3["Neo4jDriver → 实时写入"]
        W4["NetworkxKG → 内存图"]
    end

    %% 流程
    NG --> TE
    EG --> TE
    TE --> N1 & E1

    N1 --> SM
    SM -->|Yes| SM_Y --> N2
    SM -->|No| SM_N --> N2
    N2 --> N3
    N3 -->|Yes| N4 --> N5
    N3 -->|No| N6

    E1 --> E2 --> E3
    E3 -->|Yes| E4 --> E5
    E5 -->|edge| E6
    E5 -->|node| E7
    E3 -->|No| E8

    N4 --> F1
    F1 -->|Yes| F2 --> F3 --> F4
    F1 -->|No| F5

    N5 --> D1
    E6 --> D1
    E7 --> D1
    D1 -->|No| D2 --> W1 & W2 & W3 & W4
    D1 -->|Yes| D3

    %% 样式
    classDef pass fill:#c8e6c9,stroke:#2e7d32
    classDef fail fill:#ffcdd2,stroke:#c62828
    classDef process fill:#e3f2fd,stroke:#0d47a1

    class N5,E6,E7,D2 pass
    class N6,E8,D3,SM_Y fail
    class N1,N2,N4,E1,E2,E4,TE process
```

## 四、BioChatter 查询管线详细图

```mermaid
sequenceDiagram
    actor User as 用户
    participant Conv as Conversation.query()
    participant IC as _inject_context()
    participant RAS as RagAgentSelector<br/>(可选)
    participant RA as RagAgent
    participant DA as DatabaseAgent
    participant PE as BioCypherPromptEngine
    participant LLM as LLM API
    participant Neo4j as Neo4j DB
    participant PQ as _primary_query()

    User->>Conv: query("What proteins interact with p53?")
    Conv->>Conv: append_user_message(text)
    Conv->>IC: _inject_context(text)

    alt use_ragagent_selector = True
        IC->>RAS: execute(text)
        RAS->>LLM: 哪个 agent 最适合回答?
        LLM-->>RAS: "kg" (选择知识图谱)
        RAS->>RA: generate_responses(text)
    else 遍历所有 agents
        IC->>RA: generate_responses(text)
    end

    RA->>DA: get_query_results(text, k=3)

    alt use_reflexion = False (标准4步)
        DA->>PE: generate_query(text)
        PE->>LLM: Step1: 从 [Protein, Gene, Disease...] 中选实体
        LLM-->>PE: ["Protein"]
        PE->>LLM: Step2: 从 [INTERACTS_WITH, ASSOCIATED...] 中选关系
        LLM-->>PE: ["INTERACTS_WITH"]
        PE->>LLM: Step3: 选属性 [name, uniprot_id, score...]
        LLM-->>PE: {Protein: [name, uniprot_id]}
        PE->>LLM: Step4: 生成 Cypher (受 schema 约束)
        LLM-->>PE: MATCH (p1:Protein)-[r:INTERACTS_WITH]-(p2:Protein)<br/>WHERE p1.name='p53' RETURN p2 LIMIT 30
        PE-->>DA: cypher_query
        DA->>Neo4j: driver.query(cypher)
        Neo4j-->>DA: [{name:'MDM2'}, {name:'BRCA1'}, ...]
    else use_reflexion = True (反思迭代)
        DA->>PE: generate_query_prompt(text)
        PE-->>DA: query_prompt (含 schema 约束)
        DA->>DA: KGQueryReflexionAgent.execute()
        loop 迭代直到 score≥7 或有结果
            DA->>LLM: 生成/优化 Cypher
            LLM-->>DA: cypher_query
            DA->>Neo4j: 执行查询
            Neo4j-->>DA: results
            DA->>LLM: 审视结果，打分，改进
        end
    end

    DA-->>RA: List[Document]
    RA-->>IC: [(page_content, metadata)]
    IC->>Conv: append_system_message(<br/>"检索到以下信息: {results}")
    Conv->>PQ: _primary_query()
    PQ->>LLM: 用户问题 + 检索上下文 → 生成回答
    LLM-->>PQ: "p53 与 MDM2、BRCA1 等蛋白存在相互作用..."
    PQ-->>Conv: response
    Conv-->>User: (response, token_usage, correction)
```

## 五、关键模块源码位置速查

| 模块 | 文件路径 | 入口行号 | 核心职责 |
|------|---------|---------|---------|
| **BioCypher 主类** | `biocypher/_core.py` | L42 | 总编排器，懒初始化所有子模块 |
| **OntologyAdapter** | `biocypher/_ontology.py` | L30 | RDFLib解析OWL → NetworkX DiGraph |
| **Ontology** | `biocypher/_ontology.py` | (Ontology类) | Head+Tail 本体混合 + Schema扩展 |
| **OntologyMapping** | `biocypher/_mapping.py` | (类定义) | 解析 schema_config → extended_schema |
| **Translator** | `biocypher/_translate.py` | L22 | input_label映射 + 属性过滤 + 对象生成 |
| **BioCypherNode/Edge** | `biocypher/_create.py` | (数据类) | 标准化实体表示 |
| **Deduplicator** | `biocypher/_deduplicate.py` | (单例) | 全局去重 |
| **Neo4jBatchWriter** | `biocypher/output/write/graph/_neo4j.py` | - | 生成 CSV + admin import 脚本 |
| **Conversation.query()** | `biochatter/llm_connect/conversation.py` | L565 | BioChatter 唯一用户入口 |
| **_inject_context()** | `biochatter/llm_connect/conversation.py` | L1127 | RAG 上下文注入分发点 |
| **RagAgent** | `biochatter/rag_agent.py` | L15 | 统一检索封装 (KG/Vector/API) |
| **RagAgentSelector** | `biochatter/selector_agent.py` | L79 | LLM 智能选择最佳 agent |
| **DatabaseAgent** | `biochatter/database_agent.py` | L12 | KG 查询执行 (Cypher生成+Neo4j执行) |
| **BioCypherPromptEngine** | `biochatter/prompts.py` | L11 | Schema感知的4步Cypher生成 |
| **KGQueryReflexionAgent** | `biochatter/kg_langgraph_agent.py` | L91 | LangGraph 反思式查询优化 |
| **VectorDatabaseAgent** | `biochatter/vectorstore_agent.py` | L115 | Milvus 向量相似度搜索 |

## 六、逻辑架构图

纯粹的模块分区与职责边界，不含数据流箭头。展示"系统由哪些部分组成、各部分归属哪个域"。

```mermaid
block-beta
    columns 1

    block:AppLayer["应用层"]
        columns 3
        APP_WEB["Web 应用\n(Streamlit / Flask)"]
        APP_NOTEBOOK["Jupyter Notebook\n交互式分析"]
        APP_PIPELINE["自动化 Pipeline\n批量建图脚本"]
    end

    block:BioChatter["BioChatter — 对话与检索层"]
        columns 3

        block:ConvMgmt["对话管理"]:1
            columns 1
            CONV["Conversation\n会话生命周期\n消息历史 · 多轮对话 · 纠错"]
            block:LLMAdapters["LLM 适配"]
                columns 3
                LLM_OAI["OpenAI"]
                LLM_CLD["Claude"]
                LLM_GEM["Gemini"]
                LLM_OLL["Ollama"]
                LLM_AZR["Azure"]
                LLM_LIT["LiteLLM"]
            end
        end

        block:RAGOrch["RAG 编排"]:1
            columns 1
            RAG_CTRL["RAG 控制器\n_inject_context()"]
            RAG_SEL["Agent 路由\nRagAgentSelector"]
            RAG_UNI["RagAgent\n统一接口"]
        end

        block:RetrievalEngines["检索引擎"]:1
            columns 1
            R_KG["知识图谱检索\nDatabaseAgent\n· PromptEngine (Schema感知)\n· ReflexionAgent (迭代优化)"]
            R_VEC["向量检索\nVectorDatabaseAgent\n语义相似度 · 文档嵌入"]
            R_API["外部 API 检索\nAPIAgent\nBLAST · OncoKB"]
        end
    end

    block:BioCypher["BioCypher — 知识图谱构建层"]
        columns 3

        block:OntologyDomain["本体管理域"]:1
            columns 1
            O_LOAD["本体加载\nOntologyAdapter\nOWL/RDF/TTL 解析\n继承链提取 · 标签标准化\n祖先剪枝"]
            O_HYBRID["本体混合\nOntology\nHead 主骨架 (Biolink)\nTail 扩展 (Mondo/GO)\n子树嫁接 · Schema扩展"]
            O_MAP["Schema 映射\nOntologyMapping\nschema_config.yaml 解析\nextended_schema 构建\n纵向/横向/属性继承"]
        end

        block:ETLDomain["ETL 管线域"]:1
            columns 1
            ETL_TRANS["数据翻译 Translator\ninput_label 映射\n属性过滤 · 类型校验"]
            ETL_MODEL["数据模型\nBioCypherNode\nBioCypherEdge\nBioCypherRelAsNode"]
            ETL_DEDUP["去重 Deduplicator\n全局单例 · 按类型追踪"]
        end

        block:OrchDomain["编排域"]:1
            columns 1
            CORE["BioCypher 编排器\n懒初始化子模块\n统一对外 API\nwrite_nodes()\nwrite_edges()\nwrite_import_call()"]
            block:Writers["输出适配器"]
                columns 2
                W_NEO["Neo4j\nWriter"]
                W_PG["PostgreSQL\nWriter"]
                W_RDF["RDF/OWL\nWriter"]
                W_CSV["CSV/SQLite\nWriter"]
                W_NX["NetworkX\n内存"]
                W_PD["Pandas\n内存"]
            end
        end
    end

    block:StorageLayer["存储层"]
        columns 6
        NEO4J[("Neo4j")]
        PG[("PostgreSQL")]
        SQLITE[("SQLite")]
        MILVUS[("Milvus")]
        RDF_S[("RDF/OWL")]
        FS[("CSV 文件")]
    end

    block:ExtLayer["外部知识源"]
        columns 3
        BIO_DB["生物数据库\nUniProt · STRING\nReactome · DrugBank"]
        ONTO_LIB["本体库\nBiolink Model\nMondo · GO · SO"]
        LLM_SVC["LLM 服务\nGPT-4 · Claude\nGemini · Ollama"]
    end

    style AppLayer fill:#e3f2fd,stroke:#1565c0
    style BioChatter fill:#fff8e1,stroke:#f9a825
    style BioCypher fill:#e8f5e9,stroke:#2e7d32
    style StorageLayer fill:#eceff1,stroke:#546e7a
    style ExtLayer fill:#fafafa,stroke:#9e9e9e,stroke-dasharray: 5 5
    style OntologyDomain fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px
    style ETLDomain fill:#e8f5e9,stroke:#2e7d32
    style OrchDomain fill:#fffde7,stroke:#f57f17
    style ConvMgmt fill:#fff8e1,stroke:#ffa000
    style RAGOrch fill:#fff3e0,stroke:#e65100
    style RetrievalEngines fill:#fce4ec,stroke:#c62828
```

### 逻辑分层说明

| 层次 | 归属 | 职责边界 |
|------|------|---------|
| **应用层** | 用户项目 | 面向终端用户的交互入口 |
| **对话与检索层** | BioChatter | 会话管理、LLM 多厂商适配、RAG 编排、多后端检索 |
| **知识图谱构建层** | BioCypher | 本体加载混合、Schema 映射、数据翻译入库、多后端输出 |
| **存储层** | 第三方 | 图数据库、关系型数据库、向量库、语义存储、文件 |
| **外部知识源** | 第三方 | 生物数据库、标准本体、LLM 服务 |

### 域边界与职责划分

**BioCypher 三个域：**

| 域 | 模块 | 职责 |
|----|------|------|
| **本体管理域** (紫色) | OntologyAdapter、Ontology、OntologyMapping | 本体解析、混合、Schema 到本体的映射关系维护 |
| **ETL 管线域** (绿色) | Translator、DataModel、Deduplicator | 数据翻译、标准化表示、去重 |
| **编排域** (黄色) | BioCypher 主类、输出适配器 | 子模块生命周期管理、多存储后端适配 |

**BioChatter 三个域：**

| 域 | 模块 | 职责 |
|----|------|------|
| **对话管理** (橙色) | Conversation、LLM 适配器 | 会话状态、消息历史、多 LLM 厂商接入 |
| **RAG 编排** (深橙) | RAG 控制器、AgentSelector、RagAgent | 上下文注入策略、Agent 路由、统一接口封装 |
| **检索引擎** (红色) | DatabaseAgent、VectorAgent、APIAgent | KG 查询生成、向量搜索、外部 API 调用 |

### 跨层共享契约

本体管理域产出的 **`schema_info`** 是两个系统间的共享契约——BioCypher 的 ETL 域依赖它做数据翻译，BioChatter 的检索引擎域依赖它约束查询生成。这份契约确保"怎么写进去的"和"怎么查出来的"在 schema 层面严格一致。
