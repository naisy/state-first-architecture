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

### 11.2 静的解析だけでは保証できないもの

次はruntime test、contract test、property test、simulationなどが必要である。

- GuardとActionの実装内容
- 外部副作用の成功
- identityやtransactionの整合性
- 性能、メモリ、timeout
- 物理計算、数値計算、AI探索結果
- 実データに依存するlifecycle問題

SFAは「あらゆるバグを静的解析だけで保証する」とは定義しない。

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

### 14.2 Distributed SFA

Core SFAに加え、次を満たす。

- Role所有
- Boundary State
- Role間Event
- request identityまたは同等の非同期整合性契約

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
