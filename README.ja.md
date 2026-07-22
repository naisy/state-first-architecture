# State-First Architecture (SFA)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

> **システムの進行制御を、明示された State・Event・Context・Transition として表現し、状態遷移モデルの外側に隠れた制御を持たせない。**

👉 [Click here for English README (英語版はこちら)](./README.md)  
🎮 [**ブラウザで今すぐ遊ぶ：SFA ゲームデモポータル**](https://naisy.github.io/state-first-architecture/demo/)

---

## State-First Architecture（SFA）とは

State-First Architectureは、システムの制御判断を次の要素として明示的にモデル化する設計方式です。

- **State**: 制御上の現在地点
- **Event**: 状態遷移を評価する正規化された入力
- **Context**: GuardやActionが参照・更新する明示データ
- **Transition**: StateとEventを次のStateへ結ぶ規則
- **Guard**: Transitionを選択できるか判定する純粋な条件
- **Action**: 遷移に伴う変更や副作用
- **Role**: Stateや処理を所有するFrontend、Backend、Workerなどの主体

SFAが排除しようとするのは、条件分岐そのものではありません。排除対象は、状態遷移モデルの外側に置かれた、進行制御のための暗黙のフラグ、順序依存、隠れた所有権、アドホックな状態変更です。

```text
Raw Input / Timer / Network / AI Decision
                  |
                  v
             Projection
                  |
                  v
          Canonical Event
                  |
                  v
State + Context --Transition/Guard--> Action --> Next State
```

---

## SFAの中核原則

### 1. Explicit State

制御判断に影響する状態は、観測・検証可能なStateまたはContextとして明示します。進行制御に必要な情報を、状態遷移図の外側に隠しません。

### 2. Canonical Event

キー入力、API応答、タイマー、AI判断などの異なる入力源は、Projectionを通してCanonical Eventへ正規化します。その後のTransitionは入力源に依存しません。

### 3. Declared Transition

Stateの変更は、定義済みTransitionを通して行います。Frontend、Backend、Worker、AIなどがContextやStateを任意に書き換えて進行を飛び越えることはしません。

### 4. Pure Guard / Controlled Action

Guardは判定のみを担当し、Contextを変更しません。ActionはTransitionで許可された変更と副作用を担当します。

### 5. Explicit Role and Ownership

Stateを所有するRoleと、複数stepにまたがる処理のruntime ownershipを明示します。長期処理を別の判断がpreemptできるか、いつ完了・無効化・解放するかを定義します。

### 6. Observable Outcome

Event処理の結果を観測可能にします。少なくとも、正常適用、意図的破棄、契約による拒否、未処理、Action失敗を区別できるようにします。

---

## SFAが定義する範囲

SFAは、特定のUI、ゲーム、保存形式、通信方式、データベースを規定しません。次を共通仕様として扱います。

- State / Event / Context / Transition / Guard / Actionの意味
- Transition候補の評価順序
- Transitionの実行順序
- AUTO遷移の停止条件
- Projectionによる入力の正規化
- Role境界とruntime ownership
- Event outcomeと失敗の観測
- State machine instanceの生成・保持・破棄・復元に関するlifecycle境界
- 静的検証とruntime contractで確認できる範囲

一方、次はSFAを利用する各アプリケーション側の仕様です。

- ゲームで死亡後にどこへ戻るか
- F8/F9などのキー割り当て
- セーブデータのschema
- 決済、注文、装備、爆弾などのドメインルール
- timeout値、retry回数、探索budgetなどの具体値

アプリケーション固有の事例から一般原則を抽出することはできますが、事例そのものをSFAの規範仕様にはしません。

---

## SFAで得られる主な効果

### 手動操作と自動処理を同じTransitionへ合流できる

例えば、ユーザー操作と自動制御が同じCanonical Eventを発行すれば、その後の処理を複製する必要がありません。

```text
Manual Input ----\
                  > Projection --> RETURN_TO_TOWN --> shared transition
Auto Controller -/
```

### 未許可の操作をState構造で防げる

処理中Stateに`SUBMIT`のTransitionがなければ、二重送信を受理しません。`isSubmitting`のような暗黙フラグに依存せず、現在Stateが受理可能なEventを表します。

### 仕様と実装の対応を追跡しやすい

State、Event、Guard、Action、Role、Outcomeを記録することで、「なぜその処理が選ばれたか」をTransition traceとして確認できます。

### 変更の影響範囲を限定しやすい

既存TransitionとActionを維持したまま、どのEventを投影するか、どのGuardを成立させるか、どのPolicyを採用するかを変更できます。

---

## SFAが自動的に保証しないもの

SFAは、あらゆるバグを状態遷移だけで防ぐ仕組みではありません。次は別途invariant、contract test、runtime validationが必要です。

- entity identityの一意性
- 数値計算や物理計算の正しさ
- Action内部のtransaction整合性
- 永続化データの破損
- 外部APIやデバイスの失敗
- 性能、メモリ、探索時間の上限
- GuardやActionそのものの実装不良

SFAは、これらの失敗を隠さず、どのState・Event・Actionで発生したかを観測可能にするための制御構造を提供します。

---

## 概念実証（PoC）

本リポジトリには、SFAを適用した複数のゲームデモを収録しています。ゲーム固有のルールはSFA仕様ではありませんが、State、Event、Guard、Action、Projection、Role境界を確認する適用例として利用できます。

### SFA Tetris

- アニメーション中に受理しない入力をState定義で表現
- 壁キック・床キック候補を優先順位付きTransitionとして評価
- 描画とゲーム進行を分離

### SFA Freeway

- 入力と走行Stateの関係を明示
- タイトルへのescapeをEventとTransitionとして表現

### SFA Rogue

- 手動入力と自動巡回をCanonical Eventへ投影
- 複数step戦術のruntime ownershipを明示
- debug snapshotとdeterministic replayでTransitionを検証

これらはSFAの適用例であり、各ゲーム固有の仕様は技術仕様書の規範部分には含まれません。

---

## デモの動かし方

### オンラインで遊ぶ

[naisy.github.io/state-first-architecture/demo/](https://naisy.github.io/state-first-architecture/demo/)

### ローカルで実行する

1. 本リポジトリをクローンまたはダウンロードします。
2. `demo/index.html`をブラウザで開きます。

---

## ドキュメント

- [技術仕様書（日本語版）](./specification.ja.md)
- [Technical Specification（英語版）](./specification.md)

技術仕様書では、規範要件と参考実装を分離し、SFAエンジンの実行意味論、Projection、Event outcome、Role、Ownership、Lifecycle、検証可能性を定義しています。

---

## License

本プロジェクトはMITライセンスのもとで公開されています。詳細は[LICENSE](./LICENSE)をご参照ください。
