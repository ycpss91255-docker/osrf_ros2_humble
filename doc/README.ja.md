# OSRF ROS 2 Humble Docker Environment

**[English](../README.md)** | **[繁體中文](README.zh-TW.md)** | **[简体中文](README.zh-CN.md)** | **[日本語](README.ja.md)**

> **TL;DR** — ワンコマンドで `osrf/ros` desktop-full イメージベースの ROS 2 Humble コンテナ化開発環境をビルド（x86_64 のみ対応）。UID/GID を自動検出し、X11 GUI 転送に対応、マルチステージビルドで smoke test 検証を実施。
>
> ```bash
> ./build.sh && ./run.sh
> ```

---

## 目次

- [特徴](#特徴)
- [クイックスタート](#クイックスタート)
- [使い方](#使い方)
- [Subtree としての利用](#subtree-としての利用)
- [設定](#設定)
- [アーキテクチャ](#アーキテクチャ)
- [スモークテスト](#スモークテスト)
- [ディレクトリ構成](#ディレクトリ構成)
- [docker\_setup\_helper の更新](#docker_setup_helper-の更新)

---

## 特徴

- **OSRF desktop-full**：フル ROS 2 デスクトップ環境、RViz2、Gazebo 等を含む
- **マルチステージビルド**：sys → base → devel / test / runtime、用途に応じて選択
- **Smoke Test**：ビルド時に自動で Bats テストを実行し環境の正確性を検証
- **Docker Compose**：`compose.yaml` 1つで全 target を管理
- **自動検出**：`setup.sh` が UID/GID/workspace を自動検出し `.env` を生成
- **モジュール化設定**：shell config は [docker_setup_helper](https://github.com/ycpss91255-docker/docker_setup_helper) subtree で管理
- **X11 転送**：GUI アプリケーション対応（RViz2、Terminator 等）

> **注意**：このイメージは `osrf/ros` を使用しており、**x86_64** のみ対応。ARM/Raspberry Pi が必要な場合は [ros2_humble](https://github.com/ycpss91255-docker/ros2_humble) を使用してください。

## クイックスタート

```bash
# 1. 開発環境をビルド（初回は自動で .env を生成）
./build.sh

# 2. コンテナを起動
./run.sh

# 3. 起動中のコンテナに接続
./exec.sh

# または docker compose を直接使用
docker compose up -d devel
docker compose exec devel bash
docker compose down
```

## 使い方

### 開発環境（devel）

フル機能の開発環境。colcon、tmux、terminator、vim、git 等を含む。

```bash
./build.sh                       # ビルド（デフォルト devel）
./build.sh --no-env test         # ビルド（.env 更新なし）
./run.sh                         # 起動（デフォルト devel）
./run.sh --no-env -d             # バックグラウンド起動、.env 更新をスキップ
./exec.sh                        # 起動中のコンテナに接続

docker compose build devel       # 同等コマンド
docker compose run --rm devel    # ワンショット起動
docker compose up -d devel       # バックグラウンド起動
docker compose exec devel bash   # 起動中のコンテナに接続
```

### テスト（test）

ビルド時に自動で smoke test を実行。失敗するとビルドが中断される。

```bash
./build.sh test
# または
docker compose --profile test build test
```

### デプロイ（runtime）

最小化イメージ。必要な ROS 2 packages のみ含む。

```bash
./build.sh runtime
./run.sh runtime
# または
docker compose --profile runtime build runtime
docker compose --profile runtime run --rm runtime
```

## Subtree としての利用

このリポジトリは `git subtree` で他のプロジェクトに埋め込むことができ、プロジェクトに Docker 開発環境を同梱できます。

### プロジェクトへの追加

```bash
git subtree add --prefix=docker/osrf_ros2_humble \
    https://github.com/ycpss91255-docker/osrf_ros2_humble.git main --squash
```

追加後のディレクトリ構成例：

```text
my_robot_project/
├── src/                         # プロジェクトソースコード
├── docker/osrf_ros2_humble/     # Subtree
│   ├── build.sh
│   ├── run.sh
│   ├── compose.yaml
│   ├── Dockerfile
│   └── docker_setup_helper/
└── ...
```

### ビルドと実行

```bash
cd docker/osrf_ros2_humble
./build.sh && ./run.sh
```

`build.sh` は内部で `--base-path` を使用するため、どのディレクトリから実行してもパス検出が正しく動作します。

### ワークスペース検出の動作

<details>
<summary>クリックして subtree 使用時の検出動作を表示</summary>

subtree が `my_robot_project/docker/osrf_ros2_humble/` にある場合：

- **IMAGE_NAME**：ディレクトリ名は `osrf_ros2_humble`（`docker_*` ではない）ため、検出は `.env.example` の `IMAGE_NAME=osrf_ros2_humble` にフォールバック — 正常に動作。
- **WS_PATH**：戦略 1（同階層スキャン）と戦略 2（上方向走査）が一致しない場合、戦略 3（フォールバック）で親ディレクトリ（`my_robot_project/docker/`）に解決される。

**推奨**：初回ビルド後、`.env` の `WS_PATH` を実際のワークスペースに手動編集してください。以降のビルドではこの値が保持されます。

</details>

### 上流との同期

```bash
git subtree pull --prefix=docker/osrf_ros2_humble \
    https://github.com/ycpss91255-docker/osrf_ros2_humble.git main --squash
```

> **注意事項**：
> - ローカルの変更は git で通常通り追跡されます。
> - 上流があなたが変更したファイルも変更した場合、`subtree pull` で merge conflict が発生する可能性があり、手動で解決が必要です。
> - subtree 内の `docker_setup_helper/` は**直接変更しないでください** — env リポジトリ自身の subtree として管理されています。

## 設定

### .env パラメータ

`./build.sh` または `./run.sh` 実行時に自動更新（`--no-env` でスキップ）。`.env.example` を参考に手動作成も可能：

| 変数 | 説明 | 例 |
|------|------|------|
| `USER_NAME` | コンテナ内ユーザー名 | `developer` |
| `USER_GROUP` | ユーザーグループ | `developer` |
| `USER_UID` | ユーザー UID（ホストと一致） | `1000` |
| `USER_GID` | ユーザー GID（ホストと一致） | `1000` |
| `HARDWARE` | ハードウェアアーキテクチャ | `x86_64` |
| `DOCKER_HUB_USER` | Docker Hub ユーザー名 | `myuser` |
| `GPU_ENABLED` | GPU サポート | `true` / `false` |
| `IMAGE_NAME` | イメージ名 | `osrf_ros2_humble` |
| `WS_PATH` | ワークスペースマウントパス | `/home/user/colcon_ws` |

### 自動検出の詳細

`setup.sh` がシステムパラメータを自動検出し `.env` を生成。以下は2つの複雑な検出ロジックの説明。

<details>
<summary>クリックして検出ロジックを表示</summary>

#### IMAGE_NAME の推定

repo ディレクトリパスをスキャンしてイメージ名を推定：

| 優先度 | ルール | パス例 | 結果 |
|:------:|------|----------|------|
| 1 | 末尾ディレクトリが `docker_*` に一致 → プレフィックス除去 | `/home/user/docker_osrf_ros2_humble` | `osrf_ros2_humble` |
| 2 | パスを（右→左）スキャンし `*_ws` を検索 → プレフィックス取得 | `/home/user/ros2_humble_ws/docker_osrf_ros2_humble` | `ros2_humble` |
| 3 | `.env.example` の `IMAGE_NAME` を読み取り | — | `.env.example` の値 |
| 4 | フォールバック | — | `unknown` |

#### WS_PATH ワークスペース検出

3つの戦略でワークスペースマウントパスを特定：

| 優先度 | 戦略 | 条件 | 結果 |
|:------:|------|------|------|
| 1 | 同階層スキャン | カレントディレクトリが `docker_*` かつ同階層に `*_ws` あり | 同階層 `*_ws` の絶対パス |
| 2 | 上方探索 | パスを上方にたどり最初の `*_ws` コンポーネントを検索 | その `*_ws` ディレクトリ |
| 3 | フォールバック | いずれにも該当しない | repo の親ディレクトリ |

**例**（戦略 1）：
```
/home/user/
├── docker_osrf_ros2_humble/ ← repo（カレントディレクトリ）
└── osrf_ros2_humble_ws/     ← WS_PATH として検出
```

**例**（戦略 2）：
```
/home/user/colcon_ws/src/docker_osrf_ros2_humble/
                     ↑ 上方探索で *_ws を発見
```

> `.env` が既に存在し `WS_PATH` が有効なディレクトリを指している場合、検出をスキップして既存値を保持。

</details>

### 言語設定

`setup.sh` はデフォルトで英語メッセージを表示。環境変数で中国語に切り替え可能：

```bash
# .env を再生成（中国語プロンプト）
rm .env
SETUP_LANG=zh ./build.sh
```

## アーキテクチャ

### Docker Build Stage 関係図

```mermaid
graph TD
    EXT1["bats/bats:latest"]:::external
    EXT2["alpine:latest"]:::external
    EXT3["osrf/ros:humble-desktop-full-jammy"]:::external

    EXT1 --> bats-src["bats-src"]:::tool
    EXT2 --> bats-ext["bats-extensions"]:::tool

    EXT3 --> sys["sys\nuser/group・locale・timezone"]:::stage

    sys --> base["base\nsudo・git・vim・tmux・terminator・python3・colcon..."]:::stage
    base --> devel["devel\nshell config"]:::stage

    bats-src --> test["test  ⚡ ephemeral\nsmoke_test/ 実行後に破棄"]:::ephemeral
    bats-ext --> test
    devel --> test

    sys --> runtime-base["runtime-base\nsudo・tini"]:::stage
    runtime-base --> runtime["runtime\n+ 必要な ROS 2 packages"]:::stage

    classDef external fill:#555,color:#fff,stroke:#999
    classDef tool fill:#8B6914,color:#fff,stroke:#c8960c
    classDef stage fill:#1a5276,color:#fff,stroke:#2980b9
    classDef ephemeral fill:#6e2c00,color:#fff,stroke:#e67e22,stroke-dasharray:5 5
```

### Stage 説明

| Stage | FROM | 用途 |
|-------|------|------|
| `bats-src` | `bats/bats:latest` | bats バイナリソース、出荷対象外 |
| `bats-extensions` | `alpine:latest` | bats-support、bats-assert、出荷対象外 |
| `sys` | `osrf/ros:humble-desktop-full-jammy` | OS 基盤：user/group、locale、timezone |
| `base` | `sys` | 汎用開発ツール + colcon + ros2cli（apt） |
| `devel` | `base` | フル開発環境、shell 設定含む |
| `test` | `devel` | bats を注入、smoke_test/ を実行、ビルド後に破棄 |
| `runtime-base` | `sys` | 最小化 runtime ベース、dev tools なし |
| `runtime` | `runtime-base` | アプリに必要な ROS 2 packages を追加 |

## スモークテスト

`test/smoke_test/` に配置。`docker build --target test` 時に自動実行、全 **45** 項目。

<details>
<summary>クリックしてテスト詳細を表示</summary>

#### ROS 環境 (8)

| テスト項目 | 説明 |
|----------|------|
| `ROS_DISTRO` | 値が `humble` |
| `setup.bash` | ファイルが存在 |
| `setup.bash` | source 可能 |
| `ros2` | ROS を source 後に CLI 利用可能 |
| `ros2 --help` | 正常に動作 |
| `colcon` | 利用可能 |
| `colcon --help` | 正常に動作 |

#### GUI ツール (3)

| テスト項目 | 説明 |
|----------|------|
| `rviz2` | ROS を source 後に利用可能（desktop-full） |
| `rqt` | ROS を source 後に利用可能 |
| `gazebo` | 利用可能 |

#### 基本ツール (11)

| テスト項目 | 説明 |
|----------|------|
| `python3` | 利用可能 |
| `pip3` | 利用可能 |
| `git` | 利用可能 |
| `vim` | 利用可能 |
| `curl` | 利用可能 |
| `wget` | 利用可能 |
| `tmux` | 利用可能 |
| `tree` | 利用可能 |
| `htop` | 利用可能 |
| `sudo` | 利用可能 |
| `sudo` | パスワードなしで実行可能 |

#### システム (11)

| テスト項目 | 説明 |
|----------|------|
| ユーザー | root 以外 |
| `HOME` | 設定済みかつ存在 |
| タイムゾーン | `Asia/Taipei` |
| `LANG` | `en_US.UTF-8` |
| `LC_ALL` | `en_US.UTF-8` |
| `NVIDIA_VISIBLE_DEVICES` | `all` |
| `NVIDIA_DRIVER_CAPABILITIES` | `all` |
| `entrypoint.sh` | 存在かつ実行可能 |
| work ディレクトリ | 存在 |
| work ディレクトリ | 書き込み可能 |
| `bash-completion` | インストール済み |

#### スクリプト help (12)

| テスト項目 | 説明 |
|----------|------|
| `build.sh -h` | 終了コード 0 |
| `build.sh --help` | 終了コード 0 |
| `build.sh -h` | usage を表示 |
| `run.sh -h` | 終了コード 0 |
| `run.sh --help` | 終了コード 0 |
| `run.sh -h` | usage を表示 |
| `exec.sh -h` | 終了コード 0 |
| `exec.sh --help` | 終了コード 0 |
| `exec.sh -h` | usage を表示 |
| `stop.sh -h` | 終了コード 0 |
| `stop.sh --help` | 終了コード 0 |
| `stop.sh -h` | usage を表示 |

</details>

## ディレクトリ構成

```text
osrf_ros2_humble/
├── compose.yaml                 # Docker Compose 定義
├── Dockerfile                   # マルチステージビルド
├── build.sh                     # ビルドスクリプト（任意ディレクトリから実行可能）
├── run.sh                       # 起動スクリプト（任意ディレクトリから実行可能）
├── exec.sh                      # 起動中のコンテナに接続
├── stop.sh                      # コンテナを停止
├── .env.example                 # 環境変数テンプレート
├── .hadolint.yaml               # Hadolint 無視ルール
├── script/
│   └── entrypoint.sh            # コンテナエントリポイント
├── doc/                         # 翻訳版 README
│   ├── README.zh-TW.md
│   ├── README.zh-CN.md
│   └── README.ja.md
├── .github/workflows/           # CI/CD
│   ├── main.yaml                # メインパイプライン
│   ├── build-worker.yaml        # Docker build + smoke test
│   └── release-worker.yaml      # GitHub Release
├── test/smoke_test/             # Bats 環境テスト
│   ├── ros_env.bats
│   ├── script_help.bats
│   └── test_helper.bash
└── docker_setup_helper/         # git subtree (v1.3.1)
    └── src/
        ├── setup.sh             # システム検出 + .env 生成
        └── config/              # shell/pip/terminator/tmux 設定
```

## docker_setup_helper の更新

```bash
git subtree pull --prefix=docker_setup_helper \
    https://github.com/ycpss91255-docker/docker_setup_helper.git v1.3.1 --squash
```
