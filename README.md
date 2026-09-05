# cloud-itonami-souji

**Physical-AI cleaning-robot design repo** (cloud-itonami / itonami.cloud).
お掃除ロボの**設計**を agent 群(governor ⊣ advisor)で行う。**設計のみ**で、
実機を駆動する op は存在しない。

- 動かし方: [docs/operator-quickstart.md](docs/operator-quickstart.md)
  (全コマンド実行済み)
- 緑が検査していないもの: [docs/adr/0001](docs/adr/0001-what-the-green-does-not-say.md)
- 正本は superproject の
  `90-docs/adr/2800010300-cloud-itonami-souji-west-registration.edn` (accepted)。
  ⚠ この README は 2026-09-05 まで「ADR-2609011520 が正本」と書いていたが、
  **その ADR は `origin/main` に存在しない** — 経緯は ADR-0001 §4

## この repo が担うもの

- 機構構成の設計提案(brush、vacuum、wheel、lidar/camera、battery の選定)
- 設計の不変条件検査(床面積当たりの清掃率、障害物回避率)
- 人・ペット接触時の安全ゲート(恒久欠落させない)
- 仮想環境での清掃シミュレーション(床面積、障害物、電池残量)

## この repo が担わないもの

- **実機への指示・実行**: op が存在しない(設計のみ)
- **物理製造**: 外部製造パートナー連携(R2 以降)
- **starlink direct 接続の実装**: 遠隔監視のみで、通信実装は他 repo に委譲
- **spectrum**: 自前では持たない(Starlink Direct wholesale 前提)

## actor 構成

- **Cleaning Robot Design Governor ⊣ Advisor**
  - advisor: 機構要素の選定提案
  - governor: 安全性とコストのゲート。:effect :propose only
  - ops(全て propose または observe): design/propose、design/validate、
    safety/gate、sim/run、bom/observe
- 実機への指示 op は**存在させない**(設計のみ。物理制御は別 repo)

## 設計の守り

- 電池容量、走行速度、清掃効率は**全て modelled ラベル付き**(実測データなし)
- 人・ペットの安全ゲートは恒久欠落させない(esim actor の SIM-swap 拒否と同構造)
- 料金・BOM 価格は書かない(未確定)

## 段階

| 段階 | 内容 |
|---|---|
| R0 | blueprint + governor/advisor actor 骨格 + test |
| R1 | 清掃シミュレーション(床面積、障害物、電池)実装 |
| R2 | BOM + 製造ドキュメント(外部製造パートナー連携) |

## 規律

- main 直 push なし。1 task = 1 branch = 1 PR
- 全数値は modelled ラベル付き(実測データはまだ存在しない)
- credential は commit しない

## License

AGPL-3.0-or-later