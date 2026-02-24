# Balls and Bricks ブロック崩しゲーム 実装計画

## Context
Unity 6 LTS (URP) の白紙プロジェクトに「Ballz」系の数字付きブロック崩しゲームを最小実装する。
ゲームスクリプトは0件のため完全新規構築。Physics2D + TextMeshPro を活用した 2D ゲーム。

### ゲームルール
- 画面下部からボールを複数発射（マウスでエイム → クリックで発射）
- ブロックに書かれた数字 = HP（その回数ヒットで消える）
- 全ボール回収後、ブロックが 1 行下降 → 最上段に新行追加
- ブロックが最下段ライン到達 → ゲームオーバー

---

## スクリプト構成（7ファイル）

```
Assets/Scripts/
├── GameConstants.cs   // 全定数・パラメータ集中管理（staticクラス）
├── GameManager.cs     // ステートマシン・ターン進行制御
├── BallLauncher.cs    // エイム・連射・ボール回収管理
├── Ball.cs            // ボール単体の物理・コールバック
├── Block.cs           // ブロック単体のHP・表示・消滅
├── BlockManager.cs    // グリッド生成・行下降・ゲームオーバー判定
└── UIManager.cs       // ターン数・ボール数表示・GameOver画面
```

---

## ゲーム状態遷移

```
[Idle] ──マウスDownで有効方向─► [Aiming] ──マウスUp─► [Launching]
  ▲                                                          │
  │                                                    全球着地
  │                                                          ▼
  │◄──────────NextTurn(行下降+新行追加)──────────[Collecting]
  │                  │
              ゲームオーバー判定
                     │
                     ▼
                [GameOver] ──Restart─► [Idle]
```

| 状態 | 処理 |
|------|------|
| Idle | ターン開始待ち、UI更新 |
| Aiming | マウス追跡・LineRendererでエイムライン表示 |
| Launching | Coroutineで LAUNCH_INTERVAL 間隔に 1球ずつ発射 |
| Collecting | 全ボールが BottomFloor Trigger に入ると回収、先頭着地 X に集合 |
| NextTurn | ブロック行下降アニメ → 最上段新行スポーン → GameOver確認 |
| GameOver | ゲームオーバーパネル表示 |

---

## Physics2D 設定

| 設定 | 値 |
|------|-----|
| Physics2D.gravity | Vector2.zero（重力なし） |
| PhysicsMaterial2D（Ball用） | Bounciness=1.0, Friction=0.0 |
| Ball Rigidbody2D | Dynamic, GravityScale=0, CollisionDetection=Continuous |
| Block Collider | BoxCollider2D のみ（Rigidbody2D不要） |
| BottomFloor | BoxCollider2D, isTrigger=true |

**速度正規化（Ball.FixedUpdate）**：反発で速度がずれるため毎フレーム `BALL_SPEED` に固定。

```csharp
_rb.linearVelocity = _rb.linearVelocity.normalized * GameConstants.BALL_SPEED;
```

**Layer設定**：Ball(8) / Block(9) / Wall(10) を追加し、Ball同士の衝突を無効化。

---

## シーン構成

```
SampleScene
├── Main Camera           → Orthographic, Size=6.5 に変更
├── --- Game ---
│   ├── GameManager.cs
│   └── UIManager.cs
├── --- Field ---
│   ├── Walls/
│   │   ├── TopWall     BoxCollider2D  (0, +6.5) size(9, 0.2)  Wall Layer
│   │   ├── LeftWall    BoxCollider2D  (-4, 0)   size(0.2, 14) Wall Layer
│   │   ├── RightWall   BoxCollider2D  (+4, 0)   size(0.2, 14) Wall Layer
│   │   └── BottomFloor BoxCollider2D  (0, -6.3) size(9, 0.2)  isTrigger Wall Layer
│   ├── BallLauncher     BallLauncher.cs + LineRenderer
│   ├── BlockContainer   BlockManager.cs  ← Blockはここ以下に生成
│   └── GameOverLine     LineRenderer（赤い警戒ライン y=-5.0）
└── Canvas (ScreenSpace-Overlay)
    ├── TurnLabel    TextMeshProUGUI
    ├── BallLabel    TextMeshProUGUI
    └── GameOverPanel (初期非表示)
        ├── GameOverText  TextMeshProUGUI
        └── RestartButton Button
```

### プレハブ

```
Assets/Prefabs/
├── BallPhysicsMaterial.asset  (Bounciness=1, Friction=0)
├── Ball.prefab   SpriteRenderer + Rigidbody2D + CircleCollider2D + Ball.cs
└── Block.prefab  SpriteRenderer + BoxCollider2D + TextMeshPro(World) + Block.cs
```

---

## 主要パラメータ（GameConstants）

| 定数 | 値 | 説明 |
|------|----|------|
| GRID_COLS | 7 | 横列数 |
| CELL_SIZE | 1.0f | セルサイズ（m） |
| BALL_SPEED | 12.0f | ボール速度 |
| BALL_RADIUS | 0.18f | ボール半径 |
| LAUNCH_INTERVAL | 0.08f | 連射間隔（秒） |
| INITIAL_BALLS | 1 | 初期ボール数 |
| BLOCKS_PER_ROW | 5 | 1行のブロック数（ランダム配置） |
| GAMEOVER_Y | -5.0f | ゲームオーバーライン Y座標 |
| BLOCK_SHIFT_DURATION | 0.3f | 行下降アニメ時間 |

---

## 実装ステップ（実行順序）

### Phase 1: スクリプト作成（Bash）
1. `Assets/Scripts/` ディレクトリ作成
2. 7つの .cs ファイルを順番に Bash で書き込み
   - GameConstants → GameManager → Ball → BallLauncher → Block → BlockManager → UIManager

### Phase 2: コンパイル確認
3. `compile` ツールで ErrorCount=0 を確認
4. エラーがあれば `get-logs(Error)` → Bash で修正 → compile を繰り返す

### Phase 3: アセット・物理設定（execute-dynamic-code）
5. PhysicsMaterial2D 作成（Bounciness=1, Friction=0）を `Assets/Prefabs/` に保存
6. `Physics2D.gravity = Vector2.zero`
7. TagManager にレイヤー追加（Ball:8, Block:9, Wall:10）
8. Layer Collision Matrix 設定（Ball同士の衝突をOFF）

### Phase 4: プレハブ作成（execute-dynamic-code）
9. Ball.prefab 作成・保存
10. Block.prefab 作成・保存（TextMeshPro世界座標 HP表示付き）

### Phase 5: シーン組み立て（execute-dynamic-code）
11. Camera → Orthographic 変換
12. Walls / BottomFloor GameObject 作成・配置
13. BallLauncher GameObject 作成（LineRenderer付き）
14. BlockContainer GameObject 作成
15. GameOverLine（LineRenderer赤ライン）作成
16. Canvas + UI要素構築
17. GameOverPanel 構築（初期非表示）
18. Inspector 参照配線（SerializedObject）
19. シーン保存（EditorSceneManager.SaveScene）

### Phase 6: 動作確認
20. `compile` で最終エラーチェック
21. `control-play-mode(Play)` で起動
22. `get-logs(Error)` で実行時エラー確認
23. `capture-window(Game)` でスクリーンショット確認
24. `control-play-mode(Stop)`

---

## 検証ポイント

- [ ] Play で起動後、ブロックが画面上部に 3 行表示される
- [ ] マウスで上方向にエイムするとガイドラインが表示される
- [ ] クリックでボールが発射され壁・ブロックで跳ね返る
- [ ] ブロックのテキスト数字がヒットごとにデクリメントされる
- [ ] HP=0 のブロックが消える
- [ ] 全ボール回収後、ブロックが1行下降して新行が追加される
- [ ] ブロックがゲームオーバーラインを越えるとGameOver画面が表示される
- [ ] Restart ボタンで再スタートできる
