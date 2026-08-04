# ステートファーストアーキテクチャ（SFA）技術仕様書

## 1. 文書の目的と適用範囲

本書は、State-First Architecture（SFA）の共通概念と実行意味論を定義する規範仕様である。

SFAは、システムの進行制御を、明示された`State`、`Event`、`Context`、`Transition`、`Guard`、`Action`および`Role`境界としてモデル化し、状態遷移モデルの外側に隠れた制御を持たせない設計方式である。

本書は次を規定する。

- State machineの構成要素
- EventをTransitionへ適用する評価規則
- Transitionの実行順序
- AUTO遷移の停止条件
- 外部入力をCanonical Eventへ変換するProjection
- Role境界とruntime ownership
- Event outcomeと失敗の扱い
- State machine lifecycleの共通原則
- 静的検証およびruntime検証の範囲

本書は、特定アプリケーションの画面、キー操作、保存形式、通信方式、ゲームルール、業務ルールを規定しない。それらはSFAを使用する各システムのドメイン仕様である。

### 1.1 規範語

本書では次の意味で使用する。

- **MUST / しなければならない**: SFA準拠のために必須
- **MUST NOT / してはならない**: SFA準拠上禁止
- **SHOULD / することを推奨する**: 合理的理由がある場合のみ逸脱可能
- **MAY / してよい**: 任意

---

## 2. SFAの中核原則

### 2.1 Explicit State

制御判断に影響する状態は、StateまたはContextとして明示されなければならない。

進行制御のための真偽値、連番、timeout待ち、所有者、pending状態などを、状態遷移モデルから観測できない暗黙変数として保持してはならない。

局所的な計算、一時変数、描画上の値までState化する必要はない。SFAが対象とするのは、システムの進行、受理可能Event、排他、所有権、再試行、完了判定に影響する制御状態である。

### 2.2 Canonical Event

State machineが受理するEventは、入力元に依存しないCanonical Eventとして定義されなければならない。

キー入力、HTTP応答、メッセージ、タイマー、Worker通知、AI判断などのraw inputは、ProjectionによってCanonical Eventへ変換する。

### 2.3 Declared Transition

Stateの変更は、定義済みTransitionを通して行わなければならない。

Role、UI、Worker、Controller、Actionは、Transitionを経由せずに制御Stateを直接飛び越えてはならない。

### 2.4 Pure Guard / Controlled Action

Guardは、現在State、Event、明示Contextを入力としてTransitionの適用可否を判定する。GuardはContextを変更してはならない。

Actionは、選択済みTransitionに伴うContext変更または外部副作用を担当する。Actionが変更可能な範囲は明示されなければならない。

### 2.5 Explicit Role and Ownership

各Stateは、そのStateを所有するRoleを明示しなければならない。

複数stepにまたがる処理や、複数の制御主体が競合する処理では、runtime ownership、preemption可否、completion条件、invalidation条件、release条件を明示しなければならない。

### 2.6 Observable Outcome

Event処理結果は観測可能でなければならない。単に「何も起きなかった」状態にせず、適用、破棄、拒否、未処理、失敗を区別できることを推奨する。

---

## 3. コアメタモデル

### 3.1 State machine instance

State machine instanceは、少なくとも次を持つ。

```text
MachineInstance = {
  current_state,
  context,
  definition,
  role,
  lifecycle_identity
}
```

- `current_state`: 現在のState ID
- `context`: GuardおよびActionが参照する明示データ
- `definition`: StateとTransitionの定義
- `role`: instanceまたはcurrent stateの所有主体
- `lifecycle_identity`: instanceを識別するIDまたは同等の境界

### 3.2 State定義

State定義は、少なくとも次を持たなければならない。

```json
{
  "state_id": "FE_IDLE",
  "role": "frontend",
  "description": "入力を受理可能な安定状態",
  "on_enter": null,
  "on_exit": null,
  "next_transitions": []
}
```

`on_enter`および`on_exit`は任意である。使用する場合、実行順序と失敗時の扱いは本書4.3に従う。

### 3.3 Transition定義

Transitionは、少なくとも次を持つ。

```json
{
  "event": "SUBMIT",
  "guard": "can_submit",
  "action": "start_submission",
  "target_state": "FE_WAITING_BE",
  "outcome_policy": "REJECT_IF_GUARD_FALSE"
}
```

- `event`: 対象Canonical Event
- `guard`: 適用可否を判定する純粋関数。省略時は常に成立
- `action`: Context変更または副作用を行う関数。任意
- `target_state`: 遷移先State
- `outcome_policy`: 不成立時のEvent outcome方針。任意だが明示を推奨

### 3.4 Context

Contextは単なるデータ置き場ではない。制御判断に使用する値は、名前、意味、所有Role、lifecycle、復元可否を明示することを推奨する。

Context内のentity identity、version、sequence、checkpointなどのinvariantは、State machine定義とは別にcontractとして定義してよい。

#### 3.4.1 Provenance（出どころ）

制御判断に使用するContext項目は、**その値を誰がどうやって作ったか（provenance）を宣言しなければならない**。
宣言には少なくとも次を含める。

- **origin**: 値の出どころの種別（観測 / 別の機械の決定 / 外部入力 / 導出計算 / 設定）
- **producer**: 実際に値を作る箇所（関数、モジュール、機械、外部境界のいずれか）
- **freshness**: いつの時点の値か（同一Transition内で確定 / 直近の観測 / 周期更新 / 起動時固定）

同じ意味を持つContext項目を、**2箇所以上で独立に計算してはならない**。
複数の利用者が必要とする場合は、単一のproducerが作った値を渡す。

GuardはPure（§2.4）でなければならないが、**Pureであることは正しさを意味しない**。
Guardが受け取る値の作り方が誤っていれば、Guardの述語が正しくても判断は誤る。
したがってGuardの述語と同じ厳密さでprovenanceを扱う。

Guard、Action、Projectionの実装は、**宣言されたproducer以外の経路でContext項目を作ってはならない**。
判断に使う値を実装内部で新たに導出する必要が生じた場合は、まず定義側にContext項目とprovenanceを追加する。

> 判断の誤りは述語よりも「述語に渡す値の作り方」に現れやすい。provenanceの宣言は、
> その作り方を検査可能にし、同じ値の二重計算（片方だけを修正する事故の原因）を構造的に禁じる。

### 3.5 定義群をSSOTとして扱う範囲

SFAにおける信頼できる仕様源は、State JSON単体ではなく、少なくとも次の整合した定義群である。

- State catalog
- Event catalog
- Transition definitions
- Guard registryと契約
- Action registryと契約
- Projection policy
- Role / ownership policy
- Lifecycle policy
- Invariant definitions
- Contract testsまたは同等の検証資産

これらの一部がコード内に実装される場合でも、相互参照可能でなければならない。

### 3.6 Guardの契約と妥当範囲

Guard registryの契約（§3.5）は、述語の名前と真偽の意味だけでは足りない。
Guardが**近似**を含む場合、**その近似を宣言しなければならない**。

近似とは、判断を有限の資源で行うために、対象の一部だけを見る、または単純化した
モデルで代表させることを指す。例として、先読みの深さの上限、標本抽出、
キャッシュした値の再利用、上限つきの探索、単一の代表値による集約がある。

宣言には少なくとも次を含める。

- **approximated**: 何を近似したか（本来見るべき対象と、実際に見た範囲）
- **valid_when**: その近似が妥当である条件
- **breaks_when**: 近似が破れる条件（**対象の性質として書く**。「稀に」ではない）
- **on_break**: 破れたときに何が起きるか（誤って通す / 誤って拒む / 判断不能）

近似が破れたときに**誤って通す**側へ倒れる Guard は、安全に関わる判断に使ってはならない。
安全に関わる判断では、近似は**誤って拒む**側へ倒れるように設計する。

`breaks_when` に書いた条件は、検証で**負の対照として実際に踏まなければならない**（§11.3）。

> 「判断の目的にはこの範囲で十分」という判断そのものが誤り得る。
> 近似を宣言しないと、その judgement は誰にも検査されないまま残る。
> 破れる条件を対象の性質として書けば、その条件が実際に成り立つかを
> 対象のデータに当てて確かめられる。

### 3.7 責務の境界（持たない責務）

State machineの責務は、**持つもの**の列挙だけでは決まらない。
ある判断を**そのmachineが行わないこと**が設計の一部である場合、
それは定義に書かれなければ次の実装者に伝わらない。

判断が複数のmachineに分かれる系では、**そのmachineが行ってはならない判断**と、
**その判断を行う場所**を宣言することを推奨する。

宣言には少なくとも次を含める。

- **judgement**: 行ってはならない判断（何を決めないか）
- **owned_by**: その判断を行うmachineと、根拠となるinvariantの識別子
- **rationale**: なぜここで行ってはならないか

同じ判断が2箇所以上で行われると、**どちらが正なのかが定義から決まらない**。
Roleの所有（§7）はStateの所有を定めるものであり、
「このmachineがしてはならない判断」を表現しない。両者は別の宣言である。

持たない責務の宣言は、静的に検査できる形にすることを推奨する。
宣言した判断に対応する述語や入力を、そのmachineの実装が参照していないことを確かめられる。

> 判断は、置ける場所があると置かれてしまう。
> 「ここで数えれば早い」「ここで判定すれば早い」という誘因は実装のたびに生じる。
> 持たないことを宣言しておくと、その誘因に対して定義が答えを持つ。
> 宣言が無い場合、同じ規則が複数箇所に写され、時間の経過とともに食い違う。

---

## 4. エンジン実行意味論

### 4.1 Event受付

SFAエンジンは、Canonical Eventを現在のMachineInstanceへ適用する。

Eventは少なくとも次の情報を持つことを推奨する。

```text
Event = {
  type,
  payload,
  source_role,
  correlation_id?,
  metadata?
}
```

`correlation_id`やtimestampは、業務要件や観測のために使用してよい。ただし、現在Stateで受理可能かという基本判断を、隠れた時系列比較だけに依存させてはならない。

### 4.2 優先順位付きTransition評価

同じEventに複数Transitionが存在する場合、定義順を優先順位として評価する。

1. `current_state`のTransitionから`event.type`に一致する候補を定義順に取得する。
2. 各候補のGuardを順に評価する。
3. 最初に成立した候補を選択する。
4. 選択後は他候補を評価しない。
5. 成立候補がない場合は、outcome policyに従って`DISCARDED`、`REJECTED`または`UNHANDLED`を返す。

Guard評価順が仕様上意味を持つため、定義順変更は挙動変更として扱わなければならない。

#### 4.2.1 同時に成立し得る候補

手順4により、選択されなかった候補は**評価も観測もされない**。
候補が互いに排他であれば、これは単に効率の話である。
しかし**同じEventに対して2つ以上の候補が同時に成立し得る**場合、
選択された候補だけが記録され、**他の契機が成立していた事実は残らない**。

同じEventの複数候補が同時に成立し得る場合、その候補には
**同時に成立し得ることを宣言しなければならない**。

宣言には少なくとも次を含める。

- **co_satisfiable_with**: 同時に成立し得る他の候補の識別子
- **why_one_is_chosen**: それでも1本だけを選ぶ理由（同じ効果である／優先が業務上決まっている等）
- **must_record**: 選ばれなかった候補のうち、成立した事実を記録しなければならないもの

宣言された候補について、エンジンまたはActionは
**成立した候補の集合をoutcomeに残せなければならない**（§6.1 / §12）。
評価順を変えずに記録だけを足せばよく、選択の意味論は変わらない。

同じ効果を持つ複数の契機を、**先に一致した1本の名前だけで記録してはならない**。
記録が1つになると、事後の分析で「他方の契機が成立していなかったのか、
成立していたが記録されなかったのか」を区別できない。

> この欠落は実装の書き癖ではなく、優先順位付き評価の帰結である。
> 仕様は他方で**診断可能性の下限**（§10.2）を要求している。
> 「原因を辿れるようにせよ」と要求しながら、実行意味論の側で
> 辿れない状態を作らないために、宣言と記録を要求する。
> 同時に成立し得ないなら宣言は不要であり、その場合この節は何も課さない。

### 4.3 Transitionの実行順序

標準実行順序は次とする。

1. Transition候補を確定する
2. source Stateの`on_exit`を実行する
3. Transition Actionを実行する
4. `current_state`をtarget Stateへ更新する
5. target Stateの`on_enter`を実行する
6. `APPLIED` outcomeを記録する
7. target StateのAUTO Transitionを評価する

実装が別の順序を採用する場合、その順序を明示し、contract testで固定しなければならない。

### 4.4 Action失敗と原子性

Action失敗時に、State更新やContext変更が部分適用されたまま成功扱いになってはならない。

実装は次のいずれかを明示する。

- **Atomic rollback**: ContextとStateを遷移前へ戻し`FAILED`
- **Failure transition**: 失敗をContextへ記録し、定義済みfailure Stateへ遷移
- **Compensating action**: 補償Actionを実行した後にfailure Stateへ遷移

外部副作用を完全にrollbackできない場合、その事実と再試行規則を明示しなければならない。

### 4.5 AUTO Transition

`AUTO`は外部入力を待たずに評価されるtransient Eventである。

AUTO評価は安定Stateに到達するまで続けてよいが、実装は次を持たなければならない。

- 1回のdispatch内における最大AUTO遷移数
- StateまたはState+Context signatureによるcycle検出
- 上限またはcycle検出時の`FAILED`もしくは専用outcome
- AUTO Action失敗時の停止規則

無制限再帰を仕様としてはならない。

---

## 5. Projection

### 5.1 定義

Projectionは、raw inputをCanonical Eventへ変換する境界である。

```text
Raw Input -> Parse -> Validate -> Project -> Canonical Event
```

Projectionは、入力元固有の形式をState machine内部へ漏らさない。

### 5.2 複数入力源の合流

ユーザー操作、自動制御、API、Workerが同じ意味の操作を要求する場合、同じCanonical Eventへprojectionすることを推奨する。

これにより、入力源ごとに業務Actionを複製せず、同じTransition、Guard、Actionを再利用できる。

### 5.3 Projectionの責務

Projectionは次を担当してよい。

- raw inputのparse
- schema validation
- Event payloadの正規化
- source Roleの付与
- 現在Stateへdispatch可能なEvent型への変換

Projectionは、Transitionを飛び越えてContextを書き換えてはならない。

---

## 6. Event outcome

### 6.1 標準outcome

SFAエンジンは、少なくとも次のoutcomeを区別することを推奨する。

- `APPLIED`: Transitionが選択・適用された
- `DISCARDED`: 現在Stateでは意図的に無視するEvent
- `REJECTED`: Eventは認識したが、Guard、ownership、precondition、policyにより拒否
- `UNHANDLED`: Event定義またはTransitionが存在しない
- `FAILED`: 選択後のAction、on_exit、on_enter、AUTO処理などで失敗

### 6.2 DiscardとRejectの違い

高速連打中の追加クリックや、既に終了した処理への古いUI入力は`DISCARDED`としてよい。

identity不整合、権限不足、所有権競合、必須precondition不成立など、callerや運用者が知るべき事象は`REJECTED`とすることを推奨する。

定義漏れの可能性があるEventは`UNHANDLED`とし、静かに破棄しないことを推奨する。

### 6.3 観測

Outcomeには、現在State、Event、選択Transition、Guard結果、Action、target State、reasonを関連付けられることを推奨する。

同時に成立し得る候補を宣言している場合（§4.2.1）、
outcomeには**その周期で成立した候補の集合**を関連付けられなければならない。
標準outcomeは選択された1本の結末を表すものであり、
**成立した契機の数を表さない**。両者を同じ欄に畳んではならない。

### 6.4 集計キーと意味の対応

outcomeや事象を集計する場合、**その集計キーが何を数えているか**を宣言することを推奨する。

- キーの名前が、数えている対象の**部分集合**しか指していないことがある
- 別の理由で増える複数の事象が、**1つのキーに合流**していることがある

いずれの場合も、キーの値を見た者は自分が何を見ているか判定できない。
キーを改名または統合する場合は、**旧キーとの対応**を宣言する。
対応が無いと、変更前後の値を比較した判断が黙って誤る。

---

## 7. Roleと境界State

### 7.1 Role所有

Stateは所有Roleを持たなければならない。Roleの例には`frontend`、`backend`、`worker`、`device`、`shared`がある。

あるRoleは、別Role所有Stateを直接変更してはならない。Role間の進行は、EventとTransitionを通して表現する。

### 7.2 Boundary State

非同期処理やRole間通信では、待機中、適用待ち、取消待ちなどのBoundary Stateを明示することを推奨する。

Boundary Stateは受理可能Eventを限定するため、二重送信や競合要求をState構造で抑制できる。

### 7.3 遅延Event

遅延Eventが到着した時点で、現在Stateに対応Transitionがなければ、そのEventはoutcome policyに従いDiscardまたはRejectされる。

ただし、異なる要求が同じStateとEvent型を共有する場合、State不一致だけでは古いEventを区別できない。必要な場合は、明示されたrequest identity、generation、correlationなどをContextとGuardの契約として使用する。

SFAはtimestampやsequenceの使用そのものを禁止しない。それらを状態遷移図の外側に隠し、唯一の制御根拠として使用することを避ける。

---

## 8. Runtime ownershipとpreemption

### 8.1 適用条件

単発Transitionで完了する処理には、追加のownership modelは不要である。

複数Eventまたは複数turnにまたがる処理、あるいは複数のPolicyが同じMachineInstanceを制御する場合は、runtime ownershipを明示しなければならない。

### 8.2 Ownership contract

Ownership contractは少なくとも次を定義する。

- owner IDまたはowner type
- ownership開始条件
- ownerが発行可能なEvent
- 他Policyによるpreemption可否
- completion条件
- invalidation条件
- failure条件
- release後の次Stateまたは再評価方針

### 8.3 Preemption

preemptionは暗黙に発生してはならない。許可する場合、preempt Event、Guard、cleanup Action、次StateをTransitionとして定義する。

---

## 9. Lifecycle

### 9.1 対象

Lifecycleは、State machine instanceおよびそのContextの生成、開始、保持、一時停止、復元、終了、破棄を扱う。

### 9.2 Lifecycle境界

実装は次を明示することを推奨する。

- MachineInstanceの生成単位
- Context初期化のタイミング
- map、画面、session、jobなどの境界で保持するState
- transient planやfailure memoryの破棄条件
- pause/resume時に保持する情報
- checkpointに含めるState、Context、ownership
- restore時のschema validationとinvariant validation

特定のキーや保存方式はドメイン仕様である。SFAが要求するのは、復元後のStateとContextが同じ論理checkpointに属し、矛盾した組み合わせを受理しないことである。

### 9.3 ResetとResume

Reset、Resume、Retryは曖昧な直接代入ではなく、専用EventとTransitionまたは明示されたlifecycle operationとして定義することを推奨する。

---

## 10. Invariantとidentity

State machineの正しさは、Transition graphだけでは保証できない。

実装は、必要に応じて次のinvariantを定義する。

- entity identityの一意性
- sequenceの単調性
- 所有Roleの整合性
- Context内参照の存在性
- StateとContextの組み合わせ整合性
- transaction適用前後の条件
- resource上限

Invariant違反は、曖昧なActionを継続するより、`REJECTED`または`FAILED`として観測可能にすることを推奨する。

### 10.1 Invariantのscopeと関与者

Invariantは、1つのinstance内で完結するものと、**複数instanceにまたがるもの**に分かれる。
後者（相互排他、総量制限、順序関係など）は、instance単位の定義には書けない。

複数instanceにまたがるinvariantは、次を宣言しなければならない。

- **scope**: `instance` か `cross_instance` か
- **participants**: 関与するinstanceの集合をどう決めるか（判定に使う述語または索引）
- **evaluator**: 誰が検査するか（instanceではなく、集合を見られる主体）
- **cadence**: いつ検査するか（Transitionごと / 周期 / checkpoint時）

`cross_instance`のinvariant違反を記録するときは、**関与者全員のContextを含めなければならない**。
検出を起こした1つのinstanceの状態だけでは、違反が資源の重複確保によるものか、
観測の不整合によるものかを事後に判別できない。

### 10.2 Invariantの診断可能性（forensics）

各invariantは、**それが破れたときに原因を辿るために最低限必要な観測**を宣言しなければならない。
以下ではこれを**診断可能性の下限**と呼ぶ。

宣言には少なくとも次を含める。

- どのEvent／outcome／Context項目が残っていれば原因を辿れるか
- それらをどの範囲（時間、instance、資源）ぶん保持する必要があるか

観測量を抑える方針（§12）を持つ実装は、**診断可能性の下限に挙げた観測を削ってはならない**。
方針と下限は同じ場所に並べて宣言し、**両者が矛盾する場合は定義の誤りとして扱う**。

> 観測量を抑える方針は「書きすぎ」を防ぐ規律であり、「足りなさ」は防がない。
> invariantを守る責任を持つ判断が、その判断の記録を持たないまま運用に入り得る。
> 下限を先に宣言しておけば、この矛盾は運用前に検出できる。

診断可能性の下限は、invariantが**実際に破れたときにだけ意味を持つ**。
したがって平常時の観測量とは独立に決める。
平常時は集計のみで足りる観測であっても、下限に挙げたものは
**違反の検出時点で個別に記録できる形**にしておく。

### 10.3 Invariantの成立前提

invariantは、参照するContext項目の意味に依存する。
その意味が構成（設定、機能スイッチ、動作モード、縮退状態）によって変わる場合、
**invariantが成立する前提を宣言しなければならない**。

宣言には少なくとも次を含める。

- **holds_when**: そのinvariantが成立する構成の条件
- **out_of_scope_when**: 成立を主張しない構成（およびその構成で何が代わりに保証されるか）

前提を満たさない構成での違反は、**invariant違反として数えてはならない**。
「対象外」として区別できるようにする。区別しないと、本物の違反が対象外の件数に埋もれる。

同じ構成依存はGuardの`enforced`（強制するか観測のみか）にも現れる。
**invariantとGuardで前提の書き方を揃える**ことを推奨する。

> invariantが「壊れているのか、そもそもその構成では成り立たないのか」を
> 定義から判定できないと、観測された違反の意味が決まらない。
> Context項目の意味を変える変更（例: 資源の確保範囲を変える）は、
> それを参照するinvariantの見直しを伴う。前提を書いておけば、
> どのinvariantを見直すべきかが定義から辿れる。

### 10.4 進行のinvariant

§10で列挙したinvariantは、いずれも**起きてはならないこと**を述べる。
これらをすべて満たしたまま、**系が前に進まない**ことがある。
定義された遷移だけを持ち、その遷移が発火しないまま留まっている状態は、
どのinvariantにも違反していない。

進行は、単一machineの遷移graphからは決まらない。
**留まっている状態から出るEventを、そのmachine自身が作れない**ことがあるためである。
したがって進行は**宣言と検査の対象**として別に扱わなければならない。

#### 10.4.1 留まり得るsituationの宣言

**留まり得るsituation**とは、そこに入ったあと、外から何かが起きない限り出られない状態または条件を指す。

留まり得るsituationには、次を宣言しなければならない。

- **exits**: そこから出る手段。1つ以上
- **progress_measure**: 進行しているかを表す量（§10.4.3）
- **if_no_exit**: どの手段も働かなかったときに何が起きるか

宣言の置き場所は、**Stateに限らない**。
状態が変わらないまま同じ条件で留まり続ける設計では、
留まりを作っているのは**遷移を成立させないGuard**である。
その場合、宣言はGuard側に置かなければならない。
Stateだけを宣言の場所に定めると、この形の留まりは表現できない。

#### 10.4.2 脱出の提供者と、自助の条件

各`exits`には、**その手段を成立させる主体**を宣言しなければならない。

- **provider**: `self`（自分で成立させられる）／ 別のRole ／ **同じmachineの別instance**
- **waits_for**: providerが自分以外の場合、待っている相手のsituation
- **requires_change_in**: 手段の成立に**変化が必要な入力**と、その入力の所有者

`provider`が`self`であっても、`requires_change_in`の所有者が自分でない場合、
それは自助ではなく**他者への依存**である。
**同じ入力に対して同じ計算をやり直すことは、脱出ではない。**
再試行を脱出として宣言する場合は、**次に異なる結果になる理由**を書かなければならない。

自助の手段が資源の確保を必要とする場合、**その資源を誰が保持し得るか**を宣言することを推奨する。
資源が依存関係の相手に保持され得るなら、その手段はその状況では成立しない。

脱出の手段が**相手の裁量に依存する**場合、その裁量を宣言しなければならない。

- **may_be_refused**: 相手がその手段を成立させないことがあるか
- **refused_when**: 成立させない条件（相手が優先する判断、相手が守っているinvariantなど）

拒否され得る手段を、**単独の脱出として数えてはならない**。
その手段だけを`exits`に持つsituationは、宣言としては`exits`を満たしているが、
相手が拒否し続ける限り出られない。

この形は§10.4.5とは異なる。§10.4.5が扱うのは**入口が閉じている**手段、
すなわち一度も成立しない手段である。
ここで扱うのは、**成立し、Eventも届き、それでも進行しない**手段である。
到達可能性の宣言では、この形を排除できない。

拒否され得る手段しか持たない場合、次のいずれかを満たさなければならない。

- 拒否され得ない手段を1つ以上持つ
- 拒否が続いたときに何が起きるかを`if_no_exit`に書く

相手が拒否する理由は、**相手の側では正しい**ことがある。
相手が守っているinvariantが、こちらの脱出より優先される設計では、
拒否は障害ではなく宣言どおりの振る舞いである。
したがってこの形は、**相手の実装を直しても解消しない**。
解消するには、拒否され得ない手段を別に持つか、拒否された場合の帰結を引き受ける。

#### 10.4.3 進行の尺度

`progress_measure`は、次を含めて宣言しなければならない。

- **quantity**: 進行しているかを表す量
- **advances_when**: その量が進む条件
- **resets_on**: その量が振り出しに戻る条件

`resets_on`には、**系自身が進行のために行った操作を含めてはならない**。
手当てや再試行のたびに尺度が戻る設計では、
**手を打つほど、その尺度を契機とする機構が遠ざかる**。

尺度は、**進行そのものを材料に測る**ことを推奨する。
内部の分類や作業状態を材料にすると、外形が変わらないまま尺度が動いてしまう。

#### 10.4.4 依存関係の閉路

`waits_for`の宣言は、**留まり得るsituationを節点とする有限のgraph**を作る。
このgraphは静的に検査できる。

- graphに閉路がある場合、その閉路には**その閉路の中で成立し得る自助が1つ以上**存在しなければならない
- 閉路の全節点が他者を待つだけなら、その閉路に入った時点で**系は進まない**
- 自助が資源を必要とし、その資源が同じ閉路の節点に保持され得る場合、
  その自助は**その閉路については成立しない**ものとして数える

閉路そのものは禁止しない。**解ける閉路と解けない閉路を区別できることを要求する。**

#### 10.4.5 脱出の到達可能性

`exits`が定義に存在することは、**その手段が使えることを意味しない**。
別の判断がその手段の入口を閉じている場合、宣言された手段は一度も成立しない。

`exits`には、**その手段を到達可能にしている宣言**（invariantまたは同等の識別子）を
書くことを推奨する。書かれた宣言が存在しない場合、
検査はその手段を**成立しないもの**として扱う。

#### 10.4.6 検出だけを行うmachineの出口

異常や停滞を**検出するだけ**のmachineは、
**検出した事実を誰へ渡すか**を宣言しなければならない。

検出しているのに解消の帰属が無い状態は、
すべてのinvariantが満たされたまま系が進まない典型的な形である。

#### 10.4.7 解消の帰属

留まり得るsituationから出たとき、**どの手段によって出たか**を記録することを推奨する。

「出た」だけを記録すると、
機構が効いたのか、外部の事情が偶然変わったのかを事後に区別できない。
帰属の内訳は、その機構を持つ価値を測る唯一の材料になる。

出たときだけを記録する形では、**一度も出られなかった脱出について何も残らない**。
拒否され得る手段（§10.4.2）を持つsituationでは、
**その手段を試した回数**も記録することを推奨する。
試した回数が無いと、帰属の内訳は**出られた場合だけを母数とする割合**になり、
効いていない手段が「効いた実績のある手段」として残る。

#### 10.4.8 進行の前提

進行の主張は、環境に対する仮定の下でしか成り立たない。
周期処理が回り続けること、入力が届き続けることなどの仮定を
**進行の前提として宣言する**ことを推奨する。

前提が絶対時間で書かれている場合、
**実行速度が対象系の時間軸と異なる環境では意味が変わる**。
時間を含む前提は、何に対する時間かを明示する。

時間の経過を契機とする機構（周期的な見回り、静止の検出、保持期間の失効など）は、
**その経過を計る時計が進むこと**を前提に持つ。この前提は、次を宣言することを推奨する。

- **どの時計で計るか**（系の時計、実行環境が提供する時計、外部の周期など）
- **その時計が進む条件**

実行環境は、この時計の進み方を変えることがある。
背景に置かれた実行単位、資源を絞られた実行単位、
再生や早送りの下にある実行単位では、
**同じ定義が同じ速さで進むとは限らない**。

したがって、時間の経過を契機とする機構を観測して判定する場合、
**判定の前に、その時計が想定した速さで進んでいることを確かめる**ことを推奨する。

この確認を省いた観測は、**進まなかったこと**と
**進む条件が観測環境で成立していなかったこと**を区別できない。
このとき現れる誤りは、機構が動いていないという向きに出るため、
**正しい実装を変更する動機を作る**。前提の側を先に確かめないと、
実装を変更しても症状が変わらず、変更の是非も判定できない。

> 安全（起きてはならないこと）は状態空間の性質であり、遷移graphから検査できる。
> 進行（いずれ出られること）は環境の仮定に依存するため、単一machineの中では決まらない。
> 本節は進行の**証明**を求めない。求めるのは
> **誰も脱出の責務を負っていない留まりが、定義に存在しないこと**である。
> これは有限のgraph上の検査であり、実装前に行える。

---

## 11. 検証可能性

### 11.1 静的解析で確認できるもの

定義が構造化されている場合、次を静的に検出できる。

- 未定義target State
- 到達不能State
- 意図しないterminal State
- Event catalogとTransitionの不整合
- Guard / Action registryの参照切れ
- Role境界違反
- 一部のAUTO cycle
- 優先順位により到達不能なTransition
- provenanceが宣言されていないContext項目を判断に使っているTransition（§3.4.1）
- 宣言されたproducer以外がContext項目を作っている箇所（§3.4.1）
- 同じ意味のContext項目が2箇所以上で計算されていること（§3.4.1）
- 診断可能性の下限に挙げた観測が、観測量を抑える方針で削られていること（§10.2）
- `cross_instance`のinvariantにparticipants / evaluator / cadenceの宣言が無いこと（§10.1）
- 近似を含むGuardに`breaks_when`の宣言が無いこと（§3.6）
- 構成に依存するContext項目を参照するinvariantに成立前提の宣言が無いこと（§10.3）
- 同時に成立し得る候補の宣言が無いまま、同じEventの複数候補が両立し得ること（§4.2.1）
- 留まり得るsituationに`exits` / `progress_measure` / `if_no_exit`の宣言が無いこと（§10.4.1）
- `exits`の自律的な手段が時間経過だけであること（§10.4.2）
- `progress_measure`の`resets_on`に系自身の操作が含まれていること（§10.4.3）
- 依存関係のgraphの閉路に、その閉路の中で成立し得る自助が無いこと（§10.4.4）
- 検出だけを行うmachineに検出結果の受け渡し先の宣言が無いこと（§10.4.6）
- 時間の経過を契機とする機構に、計る時計とその時計が進む条件の宣言が無いこと（§10.4.8）
- 持たない責務として宣言した判断を、そのmachineの実装が参照していること（§3.7）
- 意図を壊した側の対照に、対応する診断の宣言が無いこと（§11.1.1）
- 記録が名乗るmachine名に対応する定義が存在しないこと（§12.1）
- 拒否され得る手段だけを`exits`に持つsituationに、拒否が続いた場合の帰結が無いこと（§10.4.2）
- 準拠を主張する範囲に、その範囲の母集団の宣言が無いこと（§14.4）

### 11.1.1 検査自身が満たすべき条件

静的解析は、**対象を黙って落としてはならない**。

- **扱った要素の件数**と、**扱えなかった形**を出さなければならない
- 対象が0件のときに合格を返してはならない。0件は「問題が無い」ではなく
  「まだ何も検査していない」である
- **対象が無いこと**と、**検査が対象へ到達できなかったこと**を、
  同じ結果に畳んではならない。到達できなかった場合は、合格でも不合格でもなく
  **到達不能**として報告しなければならない

対象の存在が検査の外で決まる場合（実行中の系、外部の面、生成される資料などを
対象にする場合）、検査は**対象をどう特定したか**と、
**対象が存在することを別の経路で確かめる手段**を宣言することを推奨する。

対象が無いことと到達できなかったことを畳むと、
「対象が無い」という報告は**検査の欠陥ではなく対象の不在**として読まれる。
その報告は不合格として扱われないため、
**検査されていない対象が、検査したものと同じ扱いで通過する**。

定義の保存形式や記述の形が複数ある場合、検査はそのすべてを扱うか、
**扱わなかった形を明示する**。一方の形だけを見る検査は、他方を見落としたまま合格する。

宣言を対象にする検査は、**宣言の総数**と、**検査が到達した数**を出さなければならない。
「違反0件」は、宣言のすべてを見た結果と、宣言の一部だけを見た結果を区別しない。

同じ意味の宣言が複数の置き場所に許されている場合、
到達した数は**検査が知っている置き場所の数**で決まる。
置き場所を1つしか知らない検査は、他の置き場所に置かれた宣言を
**存在しない宣言として扱う**。このとき出るのは不合格ではなく緑である。

宣言の総数は、**検査とは別の経路で数えなければならない**（§14.4）。
検査自身の探し方で総数を数えると、
到達できなかった宣言は総数からも落ちるため、到達率は常に100%になる。

### 11.1.2 検査の在庫

ある対象について、次の3つは区別できなければならない。

- **検査を当てていない**（当てる検査がまだ無い、または外している）
- **検査を当てて違反が無い**
- **検査を当てたが対象へ到達できなかった**（§11.1.1）

3つ目は検査の側の義務として§11.1.1が扱う。
1つ目は**個々の検査の結果ではなく、検査の集合の性質**であり、
どの検査の出力にも現れない。

したがって、検査を段階的に導入する場合、
**どの対象にどの検査を当てているか**を、検査の外に宣言しなければならない。
この宣言が無い場合、**まだ当てていない対象は、違反が無い対象と同じ外形を持つ**。

検査を外せる設計（対象ごとに有効・無効を切り替える設計）では、
**外している事実そのものが観測できなければならない**。
外した検査が沈黙するだけでは、外したことと合格したことが同じ結果になる。

実装の**構造**を対象にする検査（呼び出しの位置、分岐の数、記述の並びなど）は、
**その構造が何を守っているか**を検査自身に宣言しなければならない。

構造を固定する検査は、意図を変えないリファクタで失敗し得る。
宣言が無い場合、失敗した検査を弱めるしかなくなり、
**その構造が守っていた性質が黙って失われる**。

構造を固定する検査には、次の2方向の対照を用意することを推奨する。

- **構造を変えて意図を保った実装では失敗しない**
- **意図を壊した実装では失敗する**

一方だけでは、検査が守っているものを確かめられない。

意図を壊した側の対照は、**失敗したことだけでは満たされない**。
対照には、**その壊し方に対応して現れるべき診断**
（不合格の項目名、診断の識別子、失敗の種別など）を宣言しなければならない。
宣言と異なる理由で失敗した対照は、合格として扱ってはならない。

理由を問わない対照は、**検査の側の誤りによっても成立する**。
壊す手続き自体が誤っていても、対象を壊していなくても、
何らかの失敗が起きれば条件は満たされる。
その場合、対照の数は増えるが、**確かめられた性質は増えない**。

複数の対照が**すべて同じ理由で失敗する**場合、それらは独立に壊せていない。
対照の数を、確かめた範囲の広さの根拠にしてはならない。

### 11.2 静的解析だけでは保証できないもの

次はruntime test、contract test、property test、simulationなどが必要である。

- GuardとActionの実装内容
- 外部副作用の成功
- identityやtransactionの整合性
- 性能、メモリ、timeout
- 物理計算、数値計算、AI探索結果
- 実データに依存するlifecycle問題
- Guardが含む近似の妥当性（`breaks_when`が実際の対象で成り立つかは、対象のデータに当てて確かめる）
- 宣言した脱出が実際に進行を作っていること（拒否され得る手段（§10.4.2）は、
  宣言も到達可能性も満たしたまま一度も進行を作らないことがある。
  試した回数と出られた回数の比は、実行の観測でしか得られない）

SFAは「あらゆるバグを静的解析だけで保証する」とは定義しない。

エンジンが定義を実行しない場合（定義を設計と検査の基準としてのみ使う場合）、
**定義に書いたTransitionを検証側が網羅したことは、実装がそのTransitionを行う証拠にはならない**。
その場合の検査は、Transitionの網羅ではなく
Guard・invariant・機構との対応（宣言した判断がどこで行われているか）に置く。

### 11.3 推奨contract test

- State × Eventごとのoutcome
- Guard成立・不成立のA/B
- Transition実行順序
- Action失敗時のStateとContext
- AUTO上限・cycle
- Role境界
- ownership開始・完了・invalidate・preempt
- checkpoint / restore整合性
- deterministic replayが可能なシステムでは同一fixture再生
- **provenanceどおりに値が作られていること**（宣言したproducerを差し替えると判断が変わることを示す）
- **診断可能性の下限を満たしていること**（invariantを意図的に破り、宣言した観測だけで
  原因を辿れるかを確かめる。辿れなければ下限の宣言が不足している）
- **`cross_instance`のinvariant違反の記録に関与者全員のContextが含まれること**
- **Guardの`breaks_when`を実際に踏むこと**（近似が破れる条件を作り、宣言した`on_break`の
  向きへ倒れることを確かめる。踏めない場合は`breaks_when`の宣言が誤っている）
- **invariantの成立前提を満たさない構成で、違反ではなく「対象外」として扱われること**（§10.3）
- **拒否され得る脱出を実際に拒否させること**（相手が`refused_when`の条件で拒否する状況を作り、
  宣言した`if_no_exit`の帰結が現れることを確かめる。§10.4.2）
- **定義の無い名前を名乗った記録が、落とされずに名乗りだけを取り上げられること**（§12.1）

---

## 12. Traceと説明可能性

SFA実装は、Transition traceを出力できることを推奨する。

```json
{
  "from": "FE_READY",
  "event": "SUBMIT",
  "guard": "can_submit",
  "action": "start_submission",
  "to": "FE_WAITING_BE",
  "outcome": "APPLIED"
}
```

失敗時には、State、Event、Guard、Action、Role、ownership、reasonを追跡できることが望ましい。

Traceは仕様の代わりではない。定義と実行結果が一致していることを確認する観測資産である。

観測量を抑える方針（どのEventを個別に記録せず集計のみにするか、何を保持しないか）を持つ場合は、
その方針を定義側に宣言する。ただし方針は**削ってよいものだけを対象とする**。
invariantの診断可能性の下限（§10.2）に挙げた観測は方針の対象外であり、
方針と下限が矛盾する場合は定義の誤りとして扱う。

判断に使ったContextをtraceに載せる場合は、値だけでなく**provenance**（§3.4.1）を辿れることが望ましい。
値だけでは「判断が誤ったのか、判断に渡した値が誤っていたのか」を区別できない。

### 12.1 記録がmachineを名乗る場合の裏付け

記録がどのmachineのものかを名乗る場合、**その名前は定義の識別子でなければならない**。
定義の存在しない名前を名乗った記録は、名乗りを**受理してはならない**。

名乗りを受理しない場合、**記録そのものは落としてはならない。**
落とすと、その機構が動いていないことと、名乗りが裏付けを持たないことが区別できなくなる。
取り上げるのは名乗りだけであり、記録は「名乗りが裏付けを持たない」という
申告を付けて残す。

裏付けの検査は、**2つの向きを別に持たなければならない**。

- **名乗り → 定義**: 名乗った名前に対応する定義が存在すること。
  これは定義と実装の静的な検査であり、系を動かさずに行える
- **定義 → 名乗り**: 定義が観測を宣言しているのに、記録が1件も現れないこと（§10.4.6の系）。
  これは実行の観測を必要とする

2つを同じ検査に畳んではならない。畳むと、
**動かさなければ分からないもの**と**書いた時点で分かるもの**が同じ扱いになり、
静的に検出できる誤りが実行まで持ち越される。

裏付けを持たない名乗りは、**定義と記録を突き合わせる表に行を作らない**。
表の側から見ると、その名前は「観測が0件のmachine」ではなく**存在しないmachine**である。
すなわち**裏付けの無い名乗りは、記録が無いことと同じ外形を持つ**。
この形は、検査を追加するまで沈黙する。

裏付けの無い名乗りが見つかったとき、直し方は**2つある**。

- **定義を置く**: 名乗った実体が状態を持つ主体である場合
- **名乗りをやめる**: 名乗った実体が状態を持たない場合（他のmachineのContext、
  印、集計の単位など）

選択の基準は、**その実体が状態を持つ主体かどうか**である。
定義を置く側だけを正しい直し方として扱ってはならない。
状態を持たない実体に状態を与える変更は、§2.1の明示の要求を満たすように見えるが、
既存のinvariantが名前を付けて禁じた形を作ることがある（§10.3）。
この判断は§3.7（持たない責務）と同じ材料で行う。

---

## 13. 参考メタモデル

以下は参考スキーマであり、実装言語や保存形式を強制しない。

```json
{
  "state_id": "FE_READY",
  "role": "frontend",
  "description": "submitを受理可能",
  "on_enter": null,
  "on_exit": null,
  "next_transitions": [
    {
      "event": "SUBMIT",
      "guard": "can_submit",
      "action": "start_submission",
      "target_state": "FE_WAITING_BE",
      "outcome_policy": "REJECT_IF_GUARD_FALSE"
    }
  ]
}
```

Event projectionの例:

```json
{
  "raw_source": "button.click",
  "projector": "project_submit_click",
  "canonical_event": "SUBMIT"
}
```

Context項目とprovenance（§3.4.1）の例:

```json
{
  "context_item": "pending_chunk_count",
  "meaning": "未確定のchunk数",
  "origin": "observation",
  "producer": "upload_progress_reader.read()",
  "freshness": "same_transition",
  "used_by": ["can_finalize"]
}
```

Invariantの宣言（§10.1 / §10.2 / §10.3）の例:

```json
{
  "invariant_id": "one_writer_per_resource",
  "statement": "1つのresourceに対して書き込み権を持つinstanceは高々1つ",
  "scope": "cross_instance",
  "participants": "同一resource_idを保持する全instance",
  "evaluator": "resource_registry",
  "cadence": "periodic",
  "holds_when": "書き込み権の確保が単一の登録簿を経由する構成",
  "out_of_scope_when": "各instanceが独立に確保する構成（この構成では重複を検出のみ行う）",
  "forensics": {
    "required_records": [
      "書き込み権を与えたEventと、その相手（誰にいつ）",
      "違反検出時点の関与者全員のContext"
    ],
    "retention": "違反検出の前後で相手を特定できる範囲"
  }
}
```

Guardの契約（§3.6）の例:

```json
{
  "guard": "can_finalize",
  "predicate": "pending_chunk_count == 0",
  "enforced": { "strict_mode": true, "lenient_mode": "observe_only" },
  "approximation": {
    "approximated": "本来は全chunkの確定を見るが、直近の観測時点の集計だけを見る",
    "valid_when": "観測周期の間にchunkが増えない",
    "breaks_when": "観測後に新しいchunkが追加され得る構成",
    "on_break": "誤って通す"
  }
}
```

Ownership policyの例:

```json
{
  "owner": "UPLOAD_SESSION",
  "starts_on": "UPLOAD_ACCEPTED",
  "allowed_events": ["CHUNK_READY", "CANCEL", "UPLOAD_FAILED"],
  "preemptible_by": ["CANCEL"],
  "completes_on": "UPLOAD_COMPLETED",
  "invalidates_on": ["SESSION_EXPIRED", "SOURCE_CHANGED"]
}
```

持たない責務（§3.7）の例:

```json
{
  "does_not_own": [
    {
      "judgement": "資源を解放してよいかの判定",
      "owned_by": "resource_registry.one_writer_per_resource",
      "rationale": "解放の可否は資源の側で1箇所に保つ。ここで再判定すると規則が2箇所になる"
    },
    {
      "judgement": "経過時間の計測",
      "owned_by": "progress_watch.dwell_is_measured_by_position",
      "rationale": "尺度の定義を1箇所に保つ。ここで数えると尺度が2つになる"
    }
  ]
}
```

同時に成立し得る候補（§4.2.1）の例:

```json
{
  "event": "RETRY_WINDOW_ELAPSED",
  "guard": "retry_budget_left",
  "action": "discard_and_recompute",
  "target_state": "FE_RECOMPUTING",
  "co_satisfiable_with": ["escalation_selected"],
  "why_one_is_chosen": "どちらも同じ操作（現在の結果を捨てる）を行うため1本で足りる",
  "must_record": ["escalation_selected"]
}
```

留まり得るsituationの宣言（§10.4）の例:

```json
{
  "situation": "FE_WAITING_BE",
  "can_stall": true,
  "progress_measure": {
    "quantity": "確定したchunkの数",
    "advances_when": "新しいchunkが確定した",
    "resets_on": ["session_restarted"]
  },
  "exits": [
    {
      "event": "CHUNK_CONFIRMED",
      "provider": "other_role",
      "of_role": "backend",
      "waits_for": "BE_QUEUED.capacity_available",
      "may_be_refused": true,
      "refused_when": "backendが保持している整合性のinvariantを優先する場合",
      "attempts_recorded_as": "chunk_confirmation_requested"
    },
    {
      "event": "DISCARD_AND_RECOMPUTE",
      "provider": "self",
      "requires_change_in": { "what": "入力の集合", "owned_by": "self" },
      "resource": { "what": "再計算に使う作業領域", "held_by": [] },
      "reachable_because": ["upload_policy.recompute_is_always_permitted"]
    }
  ],
  "if_no_exit": "sessionは有効なまま進まず、resourceを保持し続ける"
}
```

準拠の主張の適用範囲（§14.4 / §14.5）の例:

```json
{
  "claimed_level": "core_sfa",
  "population": {
    "machines": ["upload_session", "chunk_ledger", "upload_progress_watch"],
    "enumerated_by": "定義の登録簿（読み込み時に列挙される定義の一覧）",
    "excluded": [
      {
        "machine": "legacy_transfer_queue",
        "reason": "定義はコードから逆抽出したものでSSOTがコード側にある（§3.5）"
      }
    ]
  },
  "enforced_requirements": ["explicit_state", "declared_transition", "observable_outcome"],
  "declared_but_not_enforced": [
    { "requirement": "context_provenance", "adoption": "2 / 3", "enforce_when": "3 / 3" }
  ]
}
```

構造を対象にする検査の宣言（§11.1.1）の例:

```json
{
  "check": "resource_release_is_dominated_by_the_registry",
  "target": "解放を呼ぶ全箇所",
  "protects": "解放の可否の判定が1箇所にあること",
  "pinned_structure": "解放の呼び出しより前に登録簿への問い合わせがあること",
  "controls": {
    "refactor_keeps_intent": "問い合わせを関数の入口へ移した実装では失敗しない",
    "intent_broken": "問い合わせを削った実装では失敗する",
    "intent_broken_diagnostic": "release_without_registry_lookup"
  },
  "subject_reach": {
    "identified_by": "解放を呼ぶ箇所の列挙",
    "existence_confirmed_by": "登録簿が公開する解放の口の一覧"
  }
}
```

---

## 14. 準拠レベル

### 14.1 Core SFA

次を満たす実装をCore SFA準拠とする。

- Explicit State
- Canonical Event
- Declared Transition
- Pure Guard / Controlled Action
- 定義されたTransition評価順序
- 観測可能なEvent outcome
- 判断に使用するContext項目のprovenance宣言（§3.4.1）
- Guardが近似を含む場合、その妥当範囲と破れる条件の宣言（§3.6）
- invariantを定義する場合、その診断可能性の下限の宣言（§10.2）
- invariantの成立が構成に依存する場合、その前提の宣言（§10.3）
- 同じEventの複数候補が同時に成立し得る場合、その宣言と、成立した候補を残せるoutcome（§4.2.1 / §6.3）
- 留まり得るsituationに対する`exits` / `progress_measure` / `if_no_exit`の宣言（§10.4.1）
- `exits`に対するproviderと、自助を主張する場合の`requires_change_in`の宣言（§10.4.2）

### 14.2 Distributed SFA

Core SFAに加え、次を満たす。

- Role所有
- Boundary State
- Role間Event
- request identityまたは同等の非同期整合性契約
- `cross_instance`のinvariantに対するscope / participants / evaluator / cadenceの宣言（§10.1）
- 判断が複数のmachineに分かれる場合、持たない責務とその所在の宣言（§3.7）
- 脱出の提供者が自分以外である留まりに対する`waits_for`の宣言（§10.4.2）
- 依存関係のgraphの閉路に対する、解けることの検査（§10.4.4）
- 検出だけを行うmachineに対する、検出結果の受け渡し先の宣言（§10.4.6）

### 14.3 Long-Running SFA

Core SFAに加え、次を満たす。

- runtime ownership
- completion / invalidation / preemption
- lifecycle policy
- checkpointまたはresumeを使用する場合の整合性contract

準拠レベルは優劣ではなく、システムの複雑性に応じた適用範囲を示す。

### 14.4 準拠の主張の適用範囲

準拠は**系の全体について主張するものではない**。
準拠を主張する場合、**どの範囲について主張するか**を宣言しなければならない。

範囲は、**machineの集合として列挙できなければならない**。
「この系はCore SFA準拠である」という主張は、
その系にいくつのmachineが在るかが決まっていなければ検査できない。

したがって、準拠を主張する範囲には**母集団の宣言**が必要である。

- **どのmachineが範囲に含まれるか**
- **その集合をどう決めたか**（列挙の根拠）
- **範囲の外に置いたmachineと、外した理由**

母集団の宣言は、**検査の探し方とは別に持たなければならない**（§11.1.1）。
検査が見つけた定義の集合を母集団とすると、
**見つけられなかった定義は母集団にも入らない**ため、採用率は常に100%になる。

母集団の宣言が無い場合、採用率の分母が定まらない。
このとき、宣言が満たされている件数は数えられるが、
**満たされていない件数は数えられない**。
準拠の主張は「満たしていないものが無いこと」を含むため、
分母の無い主張は検査できない。

### 14.5 段階的な導入

準拠レベルの要求を、既存の系へ一度に適用できないことがある。
このとき、次の2つは区別しなければならない。

- **要求を満たしていない**
- **要求をまだ強制していない**（宣言を集めている段階）

段階的に導入する場合、次を宣言することを推奨する。

- **いまどの要求を強制しているか**（machineごと、または範囲ごと）
- **強制していない要求について、宣言の採用率**
- **強制へ切り替える条件**

この宣言は、§11.1.2（検査の在庫）が要求する宣言と同じものでよい。
「どの要求を強制しているか」と「どの検査を当てているか」は、
同じ事実を要求の側と検査の側から述べたものである。

採用率を測らないまま要求を追加してはならない。
要求の数は、**満たされた宣言の数とは独立に増やせる**。
採用率を測らない場合、要求を追加するほど準拠から遠ざかり、
**どの要求も満たされていない状態が、要求が少ない状態と同じ外形を持つ**。

導入の段階そのものは、§2.1の明示の要求と方向が逆である。
§2.1は宣言を増やす向きに働くが、段階的な導入は
**まだ宣言を要求しない範囲を許す**。この向きを許さない場合、
既存の系は最初の宣言を書く前から全体が準拠違反となり、
**採用率という尺度が使えなくなる**。

---

## 15. 非規範の適用例

ゲーム、Web UI、分散処理、デバイス制御などは、SFAの適用例となり得る。

例えば、手動入力と自動Controllerが同じCanonical Eventへprojectionされ、既存Transitionを共有する設計は、Projectionの利点を示す。ただし、具体的なEvent名、キー、画面、保存方式、ドメインルールはSFA共通仕様ではない。

適用例は、共通仕様を説明・検証するために使用する。適用例固有のルールを、一般原則へ抽象化せずに規範仕様へ持ち込んではならない。
