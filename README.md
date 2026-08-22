# Docker Study - Next.js コンテナ化学習

## 概要

Next.js アプリケーションを題材に、Docker の基本から Dockerfile の作成、Image のビルド、Container の起動、Multi-stage Build による Image の軽量化までをハンズオン形式で学習したリポジトリです。

単に Dockerfile を作成してアプリケーションを起動するだけではなく、以下の関係や仕組みを実際に操作しながら確認しました。

- Dockerfile
- Docker Image
- Docker Container
- Docker Build
- Port Mapping
- `.dockerignore`
- Docker Layer Cache
- Multi-stage Build
- Docker と仮想マシンの違い
- CI/CD・Kubernetes とのつながり

---

# 1. 学習環境

- Windows 11
- WSL 2
- Ubuntu
- Docker Desktop
- Node.js 20
- Next.js 16
- PowerShell
- VS Code

---

# 2. 今回構築した流れ

```text
Next.js ソースコード
        ↓
Dockerfile
        ↓
docker build
        ↓
Docker Image
docker-study:latest
        ↓
docker run
        ↓
Docker Container
        ↓
Port Mapping
PC:3000 → Container:3000
        ↓
http://localhost:3000
```

最終的に、Docker Image から Container を起動し、ブラウザから Next.js アプリケーションへアクセスできることを確認しました。

---

# 3. Dockerfile / Image / Container の関係

## Dockerfile

Docker Image をどのように作るか記述する「設計書・レシピ」のようなものです。

例えば、

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./

RUN npm ci
```

のように記述します。

`docker build` を実行すると、Dockerfile に記述された命令に従って Image が作られていきます。

## Docker Image

Container を作るためのテンプレートです。

```text
Dockerfile
    ↓
docker build
    ↓
Docker Image
```

今回作成した Image は、

```text
docker-study:latest
```

です。

## Docker Container

Docker Image から作られる「実際に動く実行環境」です。

Container 自体がアプリケーションそのものというより、**アプリケーションを実際に動かすための実行環境**と理解しました。

```text
Docker Image
    ↓
docker run
    ↓
Docker Container
    ↓
Next.js 起動
```

---

# 4. Docker Image と VM Template のイメージ

これまで触れてきた VMware と比較すると、概念を理解しやすくなりました。

```text
VMware                         Docker

構築手順                   Dockerfile
   ↓                           ↓
VM Template      ≒        Docker Image
   ↓                           ↓
Virtual Machine  ≒        Docker Container
```

完全に同じ仕組みではありませんが、

- Image = テンプレート
- Container = テンプレートから作られた実行環境

という理解に役立ちました。

---

# 5. Docker と仮想マシンの違い

仮想マシンでは、それぞれの VM が Guest OS を持ちます。

```text
物理マシン
    ↓
Hypervisor
    ↓
VM
├── Guest OS
└── Application
```

一方、Container は Guest OS を丸ごと持つのではなく、ホスト側の Linux Kernel を共有します。

```text
Host
 ↓
Linux Kernel
 ↓
Container
├── Application
├── Library
└── 必要なユーザーランド
```

そのため、一般的に Container は VM と比較して、

- 軽量
- 起動が速い
- 作成・削除が容易

という特徴があります。

---

# 6. 最初に作成した Dockerfile

最初はシンプルな1ステージ構成で作成しました。

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

RUN npm run build

CMD ["npm", "start"]
```

この Dockerfile から、

```powershell
docker build -t docker-study .
```

を実行して Image を作成しました。

---

# 7. `npm run build` と `docker build` の違い

学習中に混乱したポイントの1つです。

この2つは同じ「build」という言葉を使いますが、作っているものが違います。

## npm run build

```bash
npm run build
```

これは **Next.js アプリケーションをビルドする処理**です。

```text
Next.js ソースコード
        ↓
npm run build
        ↓
.next
```

## docker build

```bash
docker build -t docker-study .
```

こちらは **Docker Image を作る処理**です。

今回の Dockerfile では、

```text
docker build
    ↓
Dockerfile を実行
    ↓
RUN npm run build
    ↓
Next.js をビルド
    ↓
必要なファイルを含める
    ↓
Docker Image 完成
```

という構造になっています。

つまり、**Docker Image をビルドする途中で、Next.js のビルドも実行している**ということです。

---

# 8. `.next` とは

`.next` は、

```bash
npm run build
```

を実行したときに Next.js が自動生成するビルド成果物です。

```text
app/
TypeScript
React
CSS
その他ソースコード
       ↓
npm run build
       ↓
.next/
```

`.next` は自分で編集するソースコードではなく、**Next.js アプリケーションを本番で動かすために生成された成果物**と理解しました。

---

# 9. Container の起動

作成した Image から Container を起動しました。

```powershell
docker run --name docker-study-container -p 3000:3000 docker-study
```

ここで、

```text
-p 3000:3000
```

によって、

```text
PC側 :3000
    ↓
Container側 :3000
```

へポートをマッピングしています。

その結果、

```text
http://localhost:3000
```

から Container 内の Next.js アプリケーションへアクセスできました。

---

# 10. Container のライフサイクル

実際に Container の起動・停止・削除を行いました。

## 起動中の Container を確認

```powershell
docker ps
```

## 停止中を含めて確認

```powershell
docker ps -a
```

## Container を停止

```powershell
docker stop docker-study-container
```

## 停止した Container を再起動

```powershell
docker start docker-study-container
```

## Container を削除

```powershell
docker rm docker-study-container
```

ここで、

```text
docker stop
→ Container は残っている
→ docker start で再起動できる

docker rm
→ Container 自体を削除する
```

という違いを確認しました。

---

# 11. Container を削除しても Image は残る

Container を削除したあと、

```powershell
docker images
```

を確認すると、

```text
docker-study:latest
```

は残っていました。

つまり、

```text
Docker Image
    ↓
Container①
    ↓
削除

Docker Imageは残る
    ↓
Container②を新しく作れる
```

という関係です。

同じ Image から何度でも Container を再作成できるため、環境を再現しやすいことを実際に確認できました。

---

# 12. `.dockerignore`

Docker Build に不要なファイルを含めないため、`.dockerignore` を作成しました。

```text
node_modules
.next
.git
.env.local
```

役割としては、

```text
.gitignore
→ Git に含めないもの

.dockerignore
→ Docker Build Context に含めないもの
```

と理解しました。

特に、

```text
node_modules
.next
```

はローカル環境で作成されたものを Image に持ち込むのではなく、**Docker 内で再現する**ようにしています。

また、

```text
.env.local
```

のように秘密情報を含む可能性のあるファイルを Image に含めないことも重要だと学びました。

---

# 13. `.dockerignore` による Build Context の削減

`.dockerignore` 作成前は、

```text
transferring context: 507.15kB
```

でした。

作成後は、

```text
transferring context: 1.25kB
```

まで減少しました。

Docker に渡す不要なファイルを除外できていることを、実際の Build Log から確認しました。

---

# 14. Docker Layer Cache

Dockerfile では、

```dockerfile
COPY package*.json ./

RUN npm ci

COPY . .
```

という順番にしています。

最初に `package.json` / `package-lock.json` だけをコピーして `npm ci` することで、依存関係に変更がなければ Docker Cache を再利用できます。

実際に2回目の Build では、

```text
CACHED COPY package*.json ./
CACHED RUN npm ci
```

となり、

```text
RUN npm ci
0.0s
```

で処理されました。

つまり、

```text
ソースコードだけ変更
        ↓
package.json は変更なし
        ↓
npm ci の結果を再利用
        ↓
Build時間を短縮
```

できます。

---

# 15. Multi-stage Build

次に、最終 Image を軽量化するため Multi-stage Build を導入しました。

最終的な Dockerfile は以下です。

```dockerfile
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

RUN npm run build


FROM node:20-alpine AS runner

WORKDIR /app

COPY --from=builder /app/package*.json ./
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public

CMD ["npm", "start"]
```

---

# 16. builder と runner とは

Dockerfile は1つですが、その中を2つの Stage に分けています。

```text
Dockerfile
│
├── builder stage
│
└── runner stage
```

## builder

builder は **アプリケーションを作る係**です。

```text
npm ci
 ↓
ソースコードをCOPY
 ↓
npm run build
 ↓
.next生成
```

ビルドに必要なファイルや依存関係は、builder 側では持っていて問題ありません。

## runner

runner は **完成したアプリケーションを動かす係**です。

builder から必要なものだけ受け取ります。

```text
builder
│
├── node_modules ───┐
├── .next ──────────┤
├── public ─────────┤
├── ソースコード     │
└── その他           │
                     ↓ 必要なものだけCOPY
runner
│
├── node_modules
├── .next
├── public
└── package関連ファイル
```

最終的に runner Stage が Docker Image になります。

---

# 17. runner でもう一度ビルドするのか？

学習中に特に疑問に感じたポイントです。

結論は、**runner では再ビルドしません。**

Next.js の Build は builder で1回だけ行います。

```text
builder
    ↓
npm run build
    ↓
.next完成
    ↓
必要なものだけrunnerへCOPY
    ↓
runner
    ↓
npm start
```

runner は builder が作った成果物を受け取り、それを実行する役割です。

---

# 18. builder と runner でデータは重複しないのか？

一部は重複します。

例えば Build 中や Docker の Build Cache 上では、

```text
builder/node_modules
runner/node_modules
```

や、

```text
builder/.next
runner/.next
```

のように、両方の Stage に同じデータが存在することがあります。

そのため、**Multi-stage Build は「ローカルPC全体のDocker使用容量を減らすための仕組み」ではない**と理解しました。

目的は、**最終的に配布・実行する Docker Image を必要最小限にすること**です。

---

# 19. なぜ Multi-stage Build を使うのか

builder はアプリケーションを作るために必要なものを持ちます。

しかし、本番でアプリケーションを動かすときには、それらすべてが必要とは限りません。

そこで、

```text
builder

ビルドに必要なもの全部
        ↓
npm run build
        ↓
成果物完成
        ↓
必要なものだけ取り出す
        ↓
runner

実行に必要なものだけ
        ↓
最終Docker Image
```

という構成にします。

つまり、

> builder はビルドするためなら重くてもよい
> runner は実行に必要なものだけを持つ

という役割分担です。

---

# 20. Image サイズの変化

Multi-stage Build 導入前：

```text
CONTENT SIZE
302MB
```

Multi-stage Build 導入後：

```text
CONTENT SIZE
190MB
```

約112MB削減できました。

```text
302MB
  ↓
190MB

約112MB削減
```

Multi-stage Build によって、最終 Image を軽量化できることを実際に確認しました。

---

# 21. Docker を使うメリット

今回の学習を通して特に理解できたのが、**同じ Image から同じ実行環境を再現しやすい**という点です。

```text
Docker Image
     ↓
開発環境でContainer起動

同じDocker Image
     ↓
検証環境でContainer起動

同じDocker Image
     ↓
本番環境でContainer起動
```

理想的には環境ごとに Image を作り直すのではなく、

```text
一度ImageをBuild
      ↓
検証
      ↓
問題なし
      ↓
同じImageを本番でも使用
```

とすることで、**「検証したもの」と「本番で動かすもの」の差を減らす**ことができます。

---

# 22. CI/CD と Docker のつながり

CI/CD と Docker を組み合わせると、

```text
Git Push
   ↓
CI
   ↓
Lint / Type Check / Test
   ↓
Docker Image Build
   ↓
Container Registry
   ↓
検証環境
   ↓
同じImage
   ↓
本番環境
```

という流れを作ることができます。

Phase 1 で学習した CI/CD と、今回の Docker がここでつながりました。

---

# 23. Kubernetes へのつながり

今回学習した、

```text
Docker Image
    ↓
Docker Container
```

という関係が、次の Kubernetes 学習の土台になります。

Docker Image から Container を作れるようになった次は、

```text
Containerを何個動かす？

1個落ちたらどうする？

アクセスをどう振り分ける？

新しいImageへどう更新する？

複数のContainerをどう管理する？
```

といった問題が出てきます。

これらを管理・自動化するのが Kubernetes / Container Orchestration の領域になります。

---

# 24. 今回理解したこと

今回のハンズオンを通して、以下を実際に操作・確認しました。

- Dockerfile は Docker Image の作り方を記述するファイル
- Docker Image は Container を作るためのテンプレート
- Docker Container は Image から作られる実行環境
- `docker build` と `npm run build` は別の処理
- `.next` は Next.js のビルド成果物
- Container を削除しても Image は残る
- 同じ Image から Container を再作成できる
- `docker stop` と `docker rm` の違い
- Port Mapping によってホストから Container へアクセスできる
- VM は Guest OS を持つが、Container はホストの Kernel を共有する
- `.dockerignore` で不要な Build Context を除外できる
- Docker Layer Cache によって Build を高速化できる
- Multi-stage Build では builder と runner に役割を分ける
- runner ではアプリケーションを再ビルドしない
- builder と runner で Build 中にデータが重複することはある
- Multi-stage Build の目的は最終 Image の軽量化
- Image サイズを 302MB → 190MB に削減できた
- 一度作成・検証した Image を複数環境で利用することで環境差を減らせる
- Docker が CI/CD と Kubernetes の間をつなぐ重要な技術であること

---

# 次の学習

## Phase 3 - Kubernetes

次は今回作成した Docker Image / Container の知識をベースに、以下を学習します。

- Pod
- Deployment
- Service
- Replica
- Scaling
- Self-healing
- Rolling Update
- Container Orchestration

```text
Phase 1
CI/CD
   ↓
Phase 2
Docker
   ↓
Phase 3
Kubernetes
```
