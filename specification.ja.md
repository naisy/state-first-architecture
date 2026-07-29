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

### 11.2 静的解析だけでは保証できないもの

次はruntime test、contract test、property test、simulationなどが必要である。

- GuardとActionの実装内容
- 外部副作用の成功
- identityやtransactionの整合性
- 性能、メモリ、timeout
- 物理計算、数値計算、AI探索結果
- 実データに依存するlifecycle問題
- Guardが含む近似の妥当性（`breaks_when`が実際の対象で成り立つかは、対象のデータに当てて確かめる）

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

### 14.2 Distributed SFA

Core SFAに加え、次を満たす。

- Role所有
- Boundary State
- Role間Event
- request identityまたは同等の非同期整合性契約
- `cross_instance`のinvariantに対するscope / participants / evaluator / cadenceの宣言（§10.1）

### 14.3 Long-Running SFA

Core SFAに加え、次を満たす。

- runtime ownership
- completion / invalidation / preemption
- lifecycle policy
- checkpointまたはresumeを使用する場合の整合性contract

準拠レベルは優劣ではなく、システムの複雑性に応じた適用範囲を示す。

---

## 15. 非規範の適用例

ゲーム、Web UI、分散処理、デバイス制御などは、SFAの適用例となり得る。

例えば、手動入力と自動Controllerが同じCanonical Eventへprojectionされ、既存Transitionを共有する設計は、Projectionの利点を示す。ただし、具体的なEvent名、キー、画面、保存方式、ドメインルールはSFA共通仕様ではない。

適用例は、共通仕様を説明・検証するために使用する。適用例固有のルールを、一般原則へ抽象化せずに規範仕様へ持ち込んではならない。
