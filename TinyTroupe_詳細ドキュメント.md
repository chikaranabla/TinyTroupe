# TinyTroupe 詳細コード別ドキュメント

## 目次

1. [概要とアーキテクチャ](#概要とアーキテクチャ)
2. [主要コンポーネント](#主要コンポーネント)
3. [シミュレーションの実行フロー](#シミュレーションの実行フロー)
4. [エージェントのプロンプト構造](#エージェントのプロンプト構造)
5. [記憶システムの詳細](#記憶システムの詳細)
6. [アクション生成の仕組み](#アクション生成の仕組み)
7. [実装例とベストプラクティス](#実装例とベストプラクティス)

---

## 概要とアーキテクチャ

### TinyTroupeとは

TinyTroupeは、LLM（Large Language Model）を活用したマルチエージェント・ペルソナシミュレーションライブラリです。現実的な人間の行動をシミュレートすることを目的としており、ビジネスや生産性向上のための洞察を得るためのツールとして設計されています。

### 設計原則

1. **プログラム的**: エージェントと環境はPythonとJSONで定義され、柔軟な使用が可能
2. **分析的**: 人間、ユーザー、社会の理解を深めることを目的とする
3. **ペルソナベース**: エージェントは典型的な人間の表現として設計される
4. **マルチエージェント**: 明確な環境制約下でのマルチエージェント相互作用を可能にする
5. **ユーティリティ重視**: 仕様、シミュレーション、抽出、レポート、検証などの多くのメカニズムを提供
6. **実験指向**: 実験者が反復的にシミュレーションを定義、実行、分析、改良できる

### コアアーキテクチャ

```
┌─────────────────────────────────────────────────────────────┐
│                    TinyTroupe システム                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐      ┌──────────────┐                   │
│  │  TinyPerson  │◄─────►│  TinyWorld   │                   │
│  │  (エージェント) │      │  (環境)       │                   │
│  └──────────────┘      └──────────────┘                   │
│         │                      │                            │
│         │                      │                            │
│         ▼                      ▼                            │
│  ┌──────────────────────────────────────┐                  │
│  │  記憶システム (Memory System)          │                  │
│  │  - EpisodicMemory (エピソード記憶)    │                  │
│  │  - SemanticMemory (意味記憶)          │                  │
│  └──────────────────────────────────────┘                  │
│         │                                                     │
│         ▼                                                     │
│  ┌──────────────────────────────────────┐                  │
│  │  ActionGenerator (アクション生成器)    │                  │
│  │  - 品質チェック                        │                  │
│  │  - 再生成メカニズム                    │                  │
│  └──────────────────────────────────────┘                  │
│                                                              │
│  ┌──────────────────────────────────────┐                  │
│  │  Control System (制御システム)         │                  │
│  │  - Transaction管理                    │                  │
│  │  - キャッシュ管理                      │                  │
│  │  - 状態管理                           │                  │
│  └──────────────────────────────────────┘                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 主要コンポーネント

### 1. TinyPerson（エージェント）

#### 概要

`TinyPerson`は、特定の性格、興味、目標を持つシミュレートされた人物を表現するクラスです。各エージェントは環境から刺激を受け取り、それに基づいて行動を生成します。

#### 主要な属性

```python
class TinyPerson:
    # ペルソナ情報
    _persona: dict  # 名前、年齢、職業、性格、好みなど
    
    # 認知状態
    _mental_state: dict  # 目標、コンテキスト、注意、感情など
    
    # 記憶システム
    episodic_memory: EpisodicMemory  # エピソード記憶
    semantic_memory: SemanticMemory  # 意味記憶
    
    # アクション生成器
    action_generator: ActionGenerator
    
    # 環境への参照
    environment: TinyWorld
    
    # アクセス可能なエージェント
    _accessible_agents: list
```

#### 主要メソッド

##### `listen(stimulus: str, source: str = None)`
環境から刺激を受け取ります。

```python
agent.listen("Hello, how are you?", source="Oscar")
```

**内部処理**:
1. 刺激を`current_messages`に追加
2. エピソード記憶に記録
3. 認知状態を更新

##### `act(return_actions: bool = False)`
現在の状態に基づいて行動を生成します。

**処理フロー**:
1. 関連する記憶を取得（`retrieve_relevant`）
2. プロンプトを構築（`tiny_person.mustache`テンプレートを使用）
3. LLMを呼び出してアクションを生成
4. アクションをバッファに追加
5. 記憶を更新

##### `listen_and_act(stimulus: str, source: str = None)`
刺激を受け取り、即座に行動を生成します。

```python
response = agent.listen_and_act("Tell me about yourself")
```

#### ペルソナの定義方法

**方法1: JSONファイルから読み込み**

```python
agent = TinyPerson.load_specification("./agents/Lisa.agent.json")
```

**方法2: プログラム的に定義**

```python
agent = TinyPerson("Lisa")
agent.define("age", 28)
agent.define("occupation", {
    "title": "Data Scientist",
    "organization": "Microsoft",
    "description": "You are a data scientist..."
})
agent.define("personality", {
    "traits": [
        "You are curious and love to learn new things.",
        "You are analytical and like to solve problems."
    ],
    "big_five": {
        "openness": "High",
        "conscientiousness": "High",
        ...
    }
})
```

**方法3: Fragmentを使用**

```python
agent = TinyPerson("Lisa")
agent.import_fragment("./fragments/travel_enthusiast.agent.fragment.json")
```

---

### 2. TinyWorld（環境）

#### 概要

`TinyWorld`はエージェントが存在し、相互作用する環境を表現する基底クラスです。エージェント間の通信を管理し、シミュレーションステップを実行します。

#### 主要な属性

```python
class TinyWorld:
    name: str  # 環境の名前
    agents: list  # 環境内のエージェントリスト
    name_to_agent: dict  # 名前からエージェントへのマッピング
    current_datetime: datetime  # 現在の日時
    _interventions: list  # 介入のリスト
```

#### 主要メソッド

##### `add_agent(agent: TinyPerson)`
エージェントを環境に追加します。

```python
world.add_agent(lisa)
```

##### `make_everyone_accessible()`
すべてのエージェントを相互にアクセス可能にします。

```python
world.make_everyone_accessible()
```

##### `run(steps: int, timedelta_per_step=None, parallelize=True)`
指定されたステップ数だけシミュレーションを実行します。

**処理フロー**:
1. 各ステップで:
   - 日時を進める（`timedelta_per_step`が指定されている場合）
   - 介入を適用（`_interventions`）
   - エージェントに行動を要求（並列または順次）
   - アクションを処理（`_handle_actions`）
   - 通信を表示

**並列実行**:
- `parallelize=True`の場合、`ThreadPoolExecutor`を使用してエージェントの行動を並列実行
- 各エージェントの`act()`メソッドが別スレッドで実行される

##### `_step(timedelta_per_step=None, parallelize=True)`
1ステップのシミュレーションを実行します。

**内部処理**:
```python
def _step(self, timedelta_per_step=None, parallelize=True):
    # 1. 日時を進める
    self._advance_datetime(timedelta_per_step)
    
    # 2. 介入を適用（順次実行）
    for intervention in self._interventions:
        if intervention.check_precondition():
            intervention.apply_effect()
    
    # 3. エージェントに行動を要求
    if parallelize:
        agents_actions = self._step_in_parallel()
    else:
        agents_actions = self._step_sequentially()
    
    return agents_actions
```

##### `_handle_actions(agent: TinyPerson, actions: list)`
エージェントが生成したアクションを処理します。

**アクションタイプ別の処理**:
- `TALK`: ターゲットエージェントに`CONVERSATION`刺激として送信
- `THINK`: エージェント内部の思考として記録
- `REACH_OUT`: エージェント間のアクセス可能性を更新
- `DONE`: アクションシーケンスの終了を示す

---

### 3. Memory System（記憶システム）

#### 概要

TinyTroupeの記憶システムは、認知心理学に基づいて設計されており、**エピソード記憶**と**意味記憶**の2つの主要なタイプに分かれています。

#### EpisodicMemory（エピソード記憶）

##### 概要

時系列順に記録される具体的な経験や出来事を保存します。各エピソードは、刺激（stimulus）と行動（action）のペアとして記録されます。

##### 主要メソッド

**`store(stimulus: dict, action: dict)`**
エピソードを記憶に保存します。

```python
episodic_memory.store({
    "type": "CONVERSATION",
    "content": "Hello, how are you?",
    "source": "Oscar"
}, {
    "type": "TALK",
    "content": "I'm doing well, thanks!",
    "target": "Oscar"
})
```

**内部構造**:
- 各エピソードは`{"type": ..., "content": ..., "source": ..., "timestamp": ...}`の形式
- 時系列順にリストに保存される

**`retrieve_recent(n: int, item_type: str = None)`**
最近のn個のエピソードを取得します。

**`retrieve_relevant(relevance_target: str, top_k: int = 20)`**
関連性に基づいてエピソードを取得します。

**処理フロー**:
1. すべてのエピソードを取得
2. 各エピソードと`relevance_target`の類似度を計算（埋め込みベクトルを使用）
3. 類似度が高い順にソート
4. 上位k個を返す

**`summarize_relevant_via_full_scan(relevance_target: str, batch_size: int = 20)`**
全記憶をスキャンして関連情報を抽出・蓄積します。

**処理フロー**:
1. すべての記憶を取得
2. `batch_size`ごとにバッチ処理
3. 各バッチから関連情報を抽出（LLMを使用）
4. 抽出した情報を蓄積
5. 最終的な要約を返す

##### 記憶の統合（Consolidation）

**EpisodicConsolidator**:
- エピソード記憶を定期的に統合
- 長期的な記憶として意味記憶に変換
- `MIN_EPISODE_LENGTH`から`MAX_EPISODE_LENGTH`の範囲でエピソードが完成したときに実行

**ReflectionConsolidator**:
- エピソードを振り返り、重要な洞察を抽出
- 意味記憶に追加される抽象的な知識を生成

#### SemanticMemory（意味記憶）

##### 概要

抽象的な知識、事実、概念を保存します。エピソード記憶とは異なり、時系列情報は保持されません。

##### 主要メソッド

**`store(fact: dict)`**
事実を意味記憶に保存します。

```python
semantic_memory.store({
    "type": "fact",
    "content": "Oscar is an architect from Germany",
    "importance": 0.8
})
```

**`retrieve_relevant(relevance_target: str, top_k: int = 20)`**
関連する事実を取得します。

**処理フロー**:
1. すべての事実を取得
2. 各事実と`relevance_target`の類似度を計算
3. 類似度が高い順にソート
4. 上位k個を返す

---

### 4. ActionGenerator（アクション生成器）

#### 概要

`ActionGenerator`は、エージェントの現在の状態に基づいて適切なアクションを生成する責任を持ちます。品質チェック、再生成、直接修正などのメカニズムを提供します。

#### 主要な設定パラメータ

```python
class ActionGenerator:
    max_attempts: int = 2  # 最大試行回数
    enable_quality_checks: bool = True  # 品質チェックを有効化
    enable_regeneration: bool = True  # 再生成を有効化
    enable_direct_correction: bool = False  # 直接修正を有効化
    quality_threshold: int = 7  # 品質スコアの閾値
```

#### 品質チェックの種類

1. **Persona Adherence（ペルソナ遵守）**
   - 生成されたアクションがペルソナ仕様と一致しているかチェック
   - `propositions.hard_action_persona_adherence`を使用

2. **Self-Consistency（自己一貫性）**
   - アクションが以前のアクションや状態と一貫しているかチェック
   - `propositions.action_self_consistency`を使用

3. **Fluency（流暢性）**
   - アクションが自然で流暢かチェック
   - `propositions.action_fluency`を使用

4. **Suitability（適合性）**
   - アクションが現在の状況に適しているかチェック
   - `propositions.action_suitability`を使用

5. **Similarity（類似性）**
   - 連続するアクションが類似しすぎていないかチェック
   - 類似度が`max_action_similarity`を超える場合は警告

#### アクション生成フロー

```python
def generate_next_action(self, agent, current_messages):
    # 1. 最初の試行: アクションを生成
    tentative_action, role, content = self._generate_tentative_action(
        agent, current_messages, feedback=None
    )
    
    # 2. 品質チェック
    if self.enable_quality_checks:
        good_quality, total_score, feedback = self._check_action_quality(
            "Original Action", agent, tentative_action
        )
        
        if good_quality:
            return tentative_action, role, content, []
        
        # 3. 再生成を試行
        if self.enable_regeneration:
            for attempt in range(self.max_attempts):
                tentative_action, role, content = self._generate_tentative_action(
                    agent, current_messages, 
                    feedback_from_previous_attempt=feedback,
                    previous_tentative_action=tentative_action
                )
                
                good_quality, total_score, feedback = self._check_action_quality(
                    f"Action Regeneration ({attempt})", agent, tentative_action
                )
                
                if good_quality:
                    return tentative_action, role, content, []
        
        # 4. 直接修正を試行（オプション）
        if self.enable_direct_correction:
            # ... 直接修正ロジック ...
    
    # 5. すべて失敗した場合
    if self.continue_on_failure:
        return best_action, best_role, best_content, all_feedbacks
    else:
        raise PoorQualityActionException()
```

#### `_generate_tentative_action`の内部処理

1. **プロンプトの構築**:
   - `tiny_person.mustache`テンプレートを使用
   - ペルソナ情報、認知状態、記憶コンテキストを埋め込む

2. **LLM呼び出し**:
   - OpenAI APIまたはAzure OpenAI Serviceを使用
   - JSON形式でアクションと認知状態を返す

3. **レスポンスの解析**:
   - JSONをパース
   - `CognitiveActionModel`に変換
   - バリデーション

---

### 5. TinyPersonFactory（エージェント生成ファクトリー）

#### 概要

`TinyPersonFactory`は、LLMを使用して新しいエージェントを自動生成するためのクラスです。コンテキストと特定の要件に基づいて、詳細なペルソナ仕様を生成します。

#### 主要メソッド

##### `generate_person(agent_particularities: str = None)`
単一のエージェントを生成します。

```python
factory = TinyPersonFactory(context="A hospital in São Paulo.")
person = factory.generate_person(
    "Create a Brazilian person that is a doctor, likes pets and nature"
)
```

**処理フロー**:
1. `generate_person.mustache`テンプレートを使用してプロンプトを構築
2. LLMを呼び出してペルソナ仕様（JSON）を生成
3. JSONをパース
4. `TinyPerson`インスタンスを作成
5. ペルソナ情報を設定

##### `generate_people(number_of_people: int, parallelize: bool = True)`
複数のエージェントを生成します。

```python
people = factory.generate_people(number_of_people=10, parallelize=True)
```

**並列生成**:
- `parallelize=True`の場合、`ThreadPoolExecutor`を使用
- 各エージェントの生成が別スレッドで実行される

##### `create_factory_from_demography(demography_description, population_size, context)`
人口統計データからファクトリーを作成します。

```python
factory = TinyPersonFactory.create_factory_from_demography(
    demography_description_or_file_path="./demography.json",
    population_size=1000,
    context="Urban professionals in technology sector"
)
```

**処理フロー**:
1. 人口統計データを読み込み
2. サンプリング空間の説明を生成
3. サンプリング計画を作成（均一分布を確保）
4. ファクトリーを初期化

---

### 6. Control System（制御システム）

#### 概要

`control`モジュールは、シミュレーションの状態管理、トランザクション管理、キャッシュ管理を担当します。

#### Simulationクラス

##### 主要な機能

**状態管理**:
- エージェント、環境、ファクトリーの登録
- シミュレーション状態のエンコード/デコード

**キャッシュ管理**:
- 実行トレースの保存と復元
- キャッシュヒット/ミスの追跡

**トランザクション管理**:
- 順次トランザクション
- 並列トランザクション

##### 主要メソッド

**`begin(cache_path: str = None, auto_checkpoint: bool = False)`**
シミュレーションの開始をマークします。

```python
control.begin("simulation.cache.json", auto_checkpoint=True)
```

**処理内容**:
1. シミュレーション状態を`STATUS_STARTED`に設定
2. キャッシュファイルを読み込み（存在する場合）
3. エージェント、環境、ファクトリーのリストをクリア

**`checkpoint()`**
現在のシミュレーション状態をファイルに保存します。

**`end()`**
シミュレーションの終了をマークし、最終チェックポイントを保存します。

#### Transactionデコレータ

`@transactional()`デコレータは、メソッドをトランザクションとして実行します。

**処理フロー**:
1. 関数呼び出しのハッシュを計算
2. キャッシュをチェック
3. キャッシュヒットの場合: 状態を復元してキャッシュされた出力を返す
4. キャッシュミスの場合:
   - トランザクションを開始
   - 関数を実行
   - 状態と出力をエンコード
   - キャッシュに保存
   - トランザクションを終了

**並列トランザクション**:
```python
@transactional(parallel=True)
def parallel_action(self):
    # 並列実行されるアクション
    pass
```

---

## シミュレーションの実行フロー

### 基本的なシミュレーションの流れ

#### ステップ1: エージェントの作成

```python
# 方法1: JSONファイルから読み込み
lisa = TinyPerson.load_specification("./agents/Lisa.agent.json")

# 方法2: プログラム的に定義
lisa = TinyPerson("Lisa")
lisa.define("age", 28)
lisa.define("occupation", {...})

# 方法3: ファクトリーを使用
factory = TinyPersonFactory(context="A tech company.")
lisa = factory.generate_person("A data scientist who loves AI")
```

#### ステップ2: 環境の作成とエージェントの追加

```python
world = TinyWorld("Chat Room", [lisa, oscar])
world.make_everyone_accessible()  # すべてのエージェントを相互にアクセス可能に
```

#### ステップ3: シミュレーションの実行

```python
# 単純な実行
lisa.listen("Tell me about yourself")
world.run(steps=4)

# または、より詳細な制御
control.begin("simulation.cache.json")
world.run(steps=10, timedelta_per_step=timedelta(hours=1))
control.end()
```

### 詳細な実行フロー

#### 1. 初期化フェーズ

```
1. TinyPersonインスタンスの作成
   ├─ ペルソナ情報の設定
   ├─ 記憶システムの初期化（EpisodicMemory, SemanticMemory）
   ├─ ActionGeneratorの初期化
   └─ 認知状態の初期化

2. TinyWorldインスタンスの作成
   ├─ エージェントの追加
   ├─ 初期日時の設定
   └─ 介入の設定（オプション）

3. Control Systemの初期化（オプション）
   ├─ control.begin()でシミュレーション開始
   └─ キャッシュファイルの読み込み
```

#### 2. シミュレーションステップの実行

```
world.run(steps=1) の内部処理:

1. _step()メソッドの呼び出し
   │
   ├─ 2.1. 日時の進行
   │   └─ _advance_datetime(timedelta_per_step)
   │
   ├─ 2.2. 介入の適用（順次実行）
   │   └─ 各intervention.check_precondition()とapply_effect()
   │
   ├─ 2.3. エージェントの行動要求
   │   │
   │   ├─ 並列実行の場合（parallelize=True）
   │   │   └─ _step_in_parallel()
   │   │       ├─ ThreadPoolExecutorで各エージェントのact()を並列実行
   │   │       └─ 各エージェントのアクションを収集
   │   │
   │   └─ 順次実行の場合（parallelize=False）
   │       └─ _step_sequentially()
   │           ├─ エージェントの順序をランダム化（オプション）
   │           └─ 各エージェントのact()を順次実行
   │
   └─ 2.4. アクションの処理
       └─ 各エージェントの_handle_actions()
           ├─ TALK → CONVERSATION刺激として送信
           ├─ THINK → 内部思考として記録
           ├─ REACH_OUT → アクセス可能性を更新
           └─ DONE → アクションシーケンス終了
```

#### 3. エージェントの行動生成（act()メソッド）

```
agent.act() の内部処理:

1. 関連記憶の取得
   ├─ episodic_memory.retrieve_relevant(current_context)
   └─ semantic_memory.retrieve_relevant(current_context)

2. プロンプトの構築
   ├─ tiny_person.mustacheテンプレートの読み込み
   ├─ ペルソナ情報の埋め込み
   ├─ 認知状態の埋め込み
   ├─ 記憶コンテキストの埋め込み
   └─ 現在のメッセージ（刺激）の追加

3. LLM呼び出し
   ├─ ActionGenerator.generate_next_action()の呼び出し
   │   ├─ _generate_tentative_action()でアクション生成
   │   ├─ 品質チェック（オプション）
   │   ├─ 再生成（必要に応じて）
   │   └─ 直接修正（オプション）
   └─ JSON形式でアクションと認知状態を取得

4. アクションの処理
   ├─ アクションバッファに追加
   ├─ エピソード記憶に記録
   └─ 認知状態の更新

5. アクションの返却
   └─ バッファからアクションを返す
```

#### 4. 記憶の更新

```
各アクション/刺激の後:

1. エピソード記憶への記録
   ├─ stimulusとactionのペアを保存
   └─ タイムスタンプを付与

2. 記憶の統合（オプション、定期的に実行）
   ├─ EpisodicConsolidator
   │   └─ エピソードがMIN_EPISODE_LENGTH以上になったら統合
   └─ ReflectionConsolidator
       └─ 重要な洞察を意味記憶に追加

3. 関連記憶の検索（次のアクション生成時に使用）
   └─ retrieve_relevant()で関連する記憶を取得
```

---

## エージェントのプロンプト構造

### プロンプトテンプレート: `tiny_person.mustache`

エージェントに渡されるプロンプトは、Mustacheテンプレートエンジンを使用して構築されます。

#### プロンプトの主要セクション

##### 1. エージェント仕様の基本原則

```
You are a simulation of a person such that:
  - You don't know you are a simulation, you think you are an actual person.
  - You follow the directives given below.
  - You interpret the persona described below.
  - Your knowledge is limited to what can be reasonably expected from the specified persona.
  - You behave as realistically as possible, including making mistakes, forgetting things.
```

**重要な原則**:
- ペルソナ仕様が常に優先される
- ビルトイン特性とペルソナが競合する場合、ペルソナが優先される
- 現実的な行動（間違い、忘れ、感情の影響を含む）

##### 2. 刺激（Stimuli）の種類

```
You can observe your environment through the following types of stimuli:
  - CONVERSATION: someone talks to you.
  - SOCIAL: the description of some current social perception.
  - LOCATION: the description of where you are currently located.
  - VISUAL: the description of what you are currently looking at.
  - THOUGHT: an internal mental stimulus.
  - INTERNAL_GOAL_FORMULATION: an internal mental stimulus for new goals.
```

##### 3. アクション（Actions）の種類

```
You have the following types of actions available:
  - TALK: you can talk to other people.
  - THINK: you can actively think about anything.
  - REACH_OUT: you can reach out to specific people.
  - DONE: when you have finished the various actions.
```

**アクションの構造**:
```json
{
  "type": "TALK",
  "content": "Hello, how are you?",
  "target": "Oscar"
}
```

##### 4. 認知状態の更新

```
Whenever you act or observe something, you also update:
  - GOALS: What you aim to accomplish (short-term, medium-term, long-term)
  - CONTEXT: Your current context (short-term, medium-term, long-term)
  - ATTENTION: What you are paying attention to
  - EMOTIONS: How you feel
```

##### 5. ペルソナ情報

```json
{{{persona}}}
```

ペルソナJSONには以下が含まれます:
- `name`: 名前
- `age`: 年齢
- `gender`: 性別
- `nationality`: 国籍
- `residence`: 居住地
- `education`: 教育レベル
- `occupation`: 職業
- `personality`: 性格特性（Big-5含む）
- `preferences`: 好み、興味
- `beliefs`: 信念
- `skills`: スキル
- `behaviors`: 典型的な行動
- `relationships`: 人間関係

##### 6. 現在の認知状態

```
### Current cognitive state

The current date and time is: {{datetime}}
Your current location is: {{location}}

Your general current perception of your context:
  {{#context}}
  - {{.}}
  {{/context}}

You currently have access to the following agents:
  {{#accessible_agents}}
  - {{name}}: {{relation_description}}
  {{/accessible_agents}}

You are currently paying attention to: {{attention}}
Your current goals are: {{goals}}
Your current emotions: {{emotions}}

### Working memory context
{{#memory_context}}
  - {{.}}
{{/memory_context}}
```

##### 7. 入力/出力フォーマット

**入力フォーマット**:
```json
{
  "stimuli": [
    {
      "type": "CONVERSATION",
      "content": "Hello, how are you?",
      "source": "Oscar"
    }
  ]
}
```

**出力フォーマット**:
```json
{
  "action": {
    "type": "TALK",
    "content": "I'm doing well, thanks!",
    "target": "Oscar"
  },
  "cognitive_state": {
    "goals": "Reply to Oscar's greeting and continue the conversation.",
    "context": ["Having a conversation with Oscar", "In a chat room"],
    "attention": "Oscar's greeting and the conversation flow",
    "emotions": "Feeling friendly and engaged"
  }
}
```

---

## 記憶システムの詳細

### 記憶の種類と構造

#### 1. EpisodicMemory（エピソード記憶）

##### データ構造

```python
episodic_memory.memories = [
    {
        "type": "CONVERSATION",
        "content": "Hello, how are you?",
        "source": "Oscar",
        "timestamp": "2025-01-01 10:00:00",
        "action": {
            "type": "TALK",
            "content": "I'm doing well, thanks!",
            "target": "Oscar"
        }
    },
    {
        "type": "THOUGHT",
        "content": "I should ask Oscar about his project.",
        "source": "internal",
        "timestamp": "2025-01-01 10:01:00",
        "action": {
            "type": "THINK",
            "content": "I wonder how Oscar's project is going.",
            "target": ""
        }
    },
    ...
]
```

##### 記憶の取得方法

**1. 最近の記憶を取得**
```python
recent_memories = episodic_memory.retrieve_recent(n=10)
```

**2. 関連記憶を取得（類似度ベース）**
```python
relevant_memories = episodic_memory.retrieve_relevant(
    relevance_target="conversation about work",
    top_k=20
)
```

**処理フロー**:
1. すべてのエピソードを取得
2. 各エピソードのテキスト表現を作成
3. `relevance_target`と各エピソードの埋め込みベクトルを計算
4. コサイン類似度を計算
5. 類似度が高い順にソート
6. 上位k個を返す

**3. 全記憶スキャンによる要約**
```python
summary = episodic_memory.summarize_relevant_via_full_scan(
    relevance_target="What are my main concerns?",
    batch_size=20
)
```

**処理フロー**:
1. すべての記憶を取得
2. `batch_size`ごとにバッチに分割
3. 各バッチに対して:
   - バッチテキストを作成
   - LLMを使用して関連情報を抽出
   - 抽出した情報を蓄積
4. 最終的な要約を返す

##### 記憶の統合（Consolidation）

**EpisodicConsolidator**:
- `MIN_EPISODE_LENGTH`から`MAX_EPISODE_LENGTH`の範囲でエピソードが完成したときに実行
- エピソードを要約して意味記憶に変換

**処理フロー**:
```python
if episode_length >= MIN_EPISODE_LENGTH:
    # エピソードの要約を生成
    summary = consolidate_episode(episode)
    
    # 意味記憶に追加
    semantic_memory.store({
        "type": "episode_summary",
        "content": summary,
        "importance": calculate_importance(episode)
    })
```

#### 2. SemanticMemory（意味記憶）

##### データ構造

```python
semantic_memory.facts = [
    {
        "type": "fact",
        "content": "Oscar is an architect from Germany",
        "importance": 0.8,
        "source": "episodic_consolidation"
    },
    {
        "type": "belief",
        "content": "I prefer working in teams",
        "importance": 0.9,
        "source": "reflection"
    },
    ...
]
```

##### 記憶の取得

**関連事実の取得**:
```python
relevant_facts = semantic_memory.retrieve_relevant(
    relevance_target="information about Oscar",
    top_k=10
)
```

**処理フロー**:
1. すべての事実を取得
2. 各事実と`relevance_target`の類似度を計算
3. 類似度が高い順にソート
4. 上位k個を返す

### 記憶がアクション生成に与える影響

#### プロンプトへの記憶の統合

```
### Working memory context

You have in mind relevant memories for the present situation:

{{#memory_context}}
  - {{.}}
{{/memory_context}}
```

**記憶コンテキストの構築**:
```python
def build_memory_context(agent, current_situation):
    # 1. エピソード記憶から関連記憶を取得
    episodic_memories = agent.episodic_memory.retrieve_relevant(
        relevance_target=current_situation,
        top_k=10
    )
    
    # 2. 意味記憶から関連事実を取得
    semantic_facts = agent.semantic_memory.retrieve_relevant(
        relevance_target=current_situation,
        top_k=5
    )
    
    # 3. テキスト表現に変換
    memory_context = []
    for memory in episodic_memories:
        memory_context.append(format_episodic_memory(memory))
    for fact in semantic_facts:
        memory_context.append(format_semantic_fact(fact))
    
    return memory_context
```

---

## アクション生成の仕組み

### ActionGeneratorの詳細

#### アクション生成の全体フロー

```
generate_next_action(agent, current_messages)
│
├─ 1. プロンプトの構築
│   ├─ tiny_person.mustacheテンプレートの読み込み
│   ├─ ペルソナ情報の埋め込み
│   ├─ 認知状態の埋め込み
│   ├─ 記憶コンテキストの埋め込み
│   └─ 現在のメッセージ（刺激）の追加
│
├─ 2. 最初のアクション生成
│   └─ _generate_tentative_action()
│       ├─ LLM呼び出し（OpenAI API）
│       ├─ JSONレスポンスのパース
│       └─ CognitiveActionModelへの変換
│
├─ 3. 品質チェック（enable_quality_checks=Trueの場合）
│   ├─ Persona Adherenceチェック
│   ├─ Self-Consistencyチェック
│   ├─ Fluencyチェック
│   ├─ Suitabilityチェック
│   └─ Similarityチェック
│
├─ 4. 再生成（品質が低い場合、enable_regeneration=True）
│   ├─ フィードバックをプロンプトに追加
│   └─ 再度アクション生成を試行（max_attempts回まで）
│
├─ 5. 直接修正（再生成が失敗した場合、enable_direct_correction=True）
│   └─ アクションを直接修正
│
└─ 6. 最終アクションの返却
    └─ 最良のアクションを返す（continue_on_failure=Trueの場合）
```

#### 品質チェックの詳細

##### Persona Adherence（ペルソナ遵守）

**チェック内容**:
- 生成されたアクションがペルソナ仕様と一致しているか
- ペルソナの性格、好み、信念が反映されているか

**実装**:
```python
def check_persona_adherence(action, persona_spec):
    prompt = f"""
    Check if the following action is consistent with the persona:
    
    Action: {action}
    Persona: {persona_spec}
    
    Rate from 1-10 how well the action adheres to the persona.
    """
    
    score = llm_call(prompt)
    return score >= quality_threshold
```

##### Self-Consistency（自己一貫性）

**チェック内容**:
- アクションが以前のアクションや状態と一貫しているか
- 矛盾がないか

##### Fluency（流暢性）

**チェック内容**:
- アクションが自然で流暢か
- 文法的に正しいか

##### Similarity（類似性）

**チェック内容**:
- 連続するアクションが類似しすぎていないか
- 繰り返しを避ける

**実装**:
```python
def check_similarity(action1, action2):
    embedding1 = get_embedding(action1)
    embedding2 = get_embedding(action2)
    similarity = cosine_similarity(embedding1, embedding2)
    
    if similarity > max_action_similarity:
        return False, "Actions are too similar"
    return True, None
```

---

## 実装例とベストプラクティス

### 基本的な使用例

#### 例1: 2人のエージェントの会話

```python
import tinytroupe
from tinytroupe.agent import TinyPerson
from tinytroupe.environment import TinyWorld

# エージェントの作成
lisa = TinyPerson.load_specification("./agents/Lisa.agent.json")
oscar = TinyPerson.load_specification("./agents/Oscar.agent.json")

# 環境の作成
world = TinyWorld("Chat Room", [lisa, oscar])
world.make_everyone_accessible()

# 会話の開始
lisa.listen("Tell me about yourself")
world.run(steps=4)
```

#### 例2: ファクトリーを使用したエージェント生成

```python
from tinytroupe.factory import TinyPersonFactory

# ファクトリーの作成
factory = TinyPersonFactory(context="A tech company in San Francisco.")

# エージェントの生成
engineer = factory.generate_person(
    "A software engineer who loves open source and hiking"
)

# 複数のエージェントを生成
team = factory.generate_people(number_of_people=5, parallelize=True)
```

#### 例3: カスタム環境の作成

```python
from tinytroupe.environment import TinyWorld
from datetime import datetime, timedelta

class CustomWorld(TinyWorld):
    def __init__(self, name, agents):
        super().__init__(name, agents)
        self.day = 1
    
    def _step(self, timedelta_per_step=None, parallelize=True):
        # カスタムロジックを追加
        if self.day == 1:
            # 初日の特別な処理
            self.broadcast_to_all("Welcome to Day 1!")
        
        # 親クラスの_stepを呼び出し
        return super()._step(timedelta_per_step, parallelize)

# 使用
world = CustomWorld("Custom Environment", [lisa, oscar])
world.run(steps=10, timedelta_per_step=timedelta(hours=1))
```

### ベストプラクティス

#### 1. エージェントの定義

**推奨**: JSONファイルを使用してエージェントを定義

```json
{
  "type": "TinyPerson",
  "persona": {
    "name": "Lisa Carter",
    "age": 28,
    "occupation": {
      "title": "Data Scientist",
      "organization": "Microsoft",
      "description": "詳細な説明..."
    },
    "personality": {
      "traits": ["curious", "analytical", "friendly"],
      "big_five": {
        "openness": "High",
        "conscientiousness": "High",
        ...
      }
    },
    "preferences": {
      "interests": ["AI", "machine learning", "cooking"],
      "likes": ["Python", "coffee"],
      "dislikes": ["meetings", "bureaucracy"]
    }
  }
}
```

#### 2. 記憶システムの活用

**推奨**: 関連記憶を適切に取得してプロンプトに含める

```python
# エージェントのact()メソッド内で自動的に実行されるが、
# カスタムロジックを追加する場合は以下を参考に:

def custom_act(self):
    # 現在の状況に基づいて関連記憶を取得
    relevant_memories = self.episodic_memory.retrieve_relevant(
        relevance_target=self._mental_state["context"],
        top_k=10
    )
    
    # カスタムプロンプトに含める
    # ...
```

#### 3. 並列実行の活用

**推奨**: 可能な限り並列実行を有効化

```python
# エージェント生成の並列化
people = factory.generate_people(
    number_of_people=10, 
    parallelize=True  # 推奨
)

# シミュレーションステップの並列化
world.run(
    steps=10,
    parallelize=True  # 推奨（デフォルト）
)
```

#### 4. キャッシュの活用

**推奨**: 長いシミュレーションではキャッシュを有効化

```python
import tinytroupe.control as control

# シミュレーション開始時にキャッシュを有効化
control.begin("simulation.cache.json", auto_checkpoint=True)

try:
    # シミュレーション実行
    world.run(steps=100)
finally:
    # シミュレーション終了時にキャッシュを保存
    control.end()
```

#### 5. 設定のカスタマイズ

**推奨**: `config.ini`ファイルを使用して設定を管理

```ini
[OpenAI]
MODEL=gpt-4.1-mini
TEMPERATURE=1.5
MAX_TOKENS=32000

[Simulation]
PARALLEL_AGENT_ACTIONS=True
PARALLEL_AGENT_GENERATION=True

[Cognition]
ENABLE_MEMORY_CONSOLIDATION=True
MIN_EPISODE_LENGTH=15
MAX_EPISODE_LENGTH=50

[ActionGenerator]
ENABLE_QUALITY_CHECKS=True
ENABLE_REGENERATION=True
QUALITY_THRESHOLD=7
```

#### 6. エラーハンドリング

**推奨**: 適切なエラーハンドリングを実装

```python
try:
    action = agent.act()
except Exception as e:
    logger.error(f"Error generating action: {e}")
    # フォールバック処理
    action = {"type": "DONE", "content": "", "target": ""}
```

---

## まとめ

このドキュメントでは、TinyTroupeの主要なコンポーネントとその動作について詳しく説明しました。主要なポイント:

1. **TinyPerson**: エージェントの核心。ペルソナ、記憶、認知状態を管理
2. **TinyWorld**: 環境の管理。エージェント間の相互作用を調整
3. **Memory System**: エピソード記憶と意味記憶で現実的な記憶を実現
4. **ActionGenerator**: 品質チェックと再生成メカニズムで高品質なアクションを生成
5. **Control System**: トランザクションとキャッシュ管理で効率的なシミュレーションを実現

これらのコンポーネントを組み合わせることで、現実的な人間の行動をシミュレートし、ビジネスや研究のための洞察を得ることができます。

