---
title: "RailwayにNextjsアプリケーションをデプロイする時のプラクティス"
emoji: "🕯"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Railway", "TypeScript", "Nextjs", "Postgres", "DrizzleORM"]
published: false
---
# はじめに
Webサービスを作成した時に、ある程度整っているPaaSであると言われていたRailwayにデプロイしました。Railwayに何かをデプロイする時の解説記事がまだ少ないので、私がデプロイした時のやり方を一例としてまとめます。

## 対象読者
- Webサービスのデプロイ・外部公開にRailwayを使いたいがやり方がよく分からない人
- Railwayへのサインアップ、クレジットカードの登録等は完了している人
- GitHubを利用し、RailwayのWeb上でデプロイしたい人

## サービスに使用している技術
- Next.js
- TypeScript
- Tailwind CSS
- Posgres SQL
- Drizzle ORM

# Railwayについて
Railwayは、シンプルなクラウドデプロイメントプラットフォームで、GitHubリポジトリと連携してアプリケーションを簡単にデプロイできます。
- GitHubとの自動連携: リポジトリの変更を自動検知してデプロイ
- 多言語対応: Node.js、Python、Go、Rust、PHP等をサポート
- データベース統合: PostgreSQL、MySQL、Redis等を簡単に追加可能
- 環境変数管理: UI上で環境変数を簡単に設定・管理
- カスタムドメイン: 独自ドメインの設定が可能

# デプロイのやり方
[Railwaiの初期画面](https://railway.com/)の右上からDashboardページに入り、"+New"ボタンを押して次の画面を用意してください。
![](/images/railway_one_practice/select_newProject.png)
デプロイしたいプロジェクトのリポジトリを選択すると、自動的に最初のビルドが開始されます。
![](/images/railway_one_practice/select_repo.png)
![](/images/railway_one_practice/start_initialBuild.png)
もし開発用とリリース用でブランチを分けている場合、最初のビルドをキャンセルして、Settingsタブからブランチを切り替えてください。
![](/images/railway_one_practice/select_deployAbort.png)
![](/images/railway_one_practice/select_repoBranch.png)

# Postgresサービスの追加
リポジトリのビルドが置かれているプロジェクトダッシュボードの画面から、右上の`+ Create`ボタンをクリックしてください。
ポップアップした画面から、`Database > Add PostgresSQL`の順にクリックすることで、ダッシュボードにPostgresのサービスが追加されます。

# Railwayビルド用のスクリプト
プロジェクトのルートディレクトリに`railway.toml`ファイルを作成し、以下の内容を記述してください。
```toml
[build]
builder = "NIXPACKS"
buildCommand = "npm run build"

[deploy]
startCommand = "npx tsx ./scripts/check-db.ts && ./scripts/railway-start.sh"
healthcheckPath = "/api/health"
healthcheckTimeout = 600
restartPolicyType = "ON_FAILURE"

[variables]
PORT = "8080"
NODE_ENV = "production"
```

また、`scripts`フォルダをプロジェクトルートに配置し、その下にDB接続チェック用の`check-db.ts`とRailway起動シェルスクリプトの`railway-start.sh`を以下のように記述して配置してください。
```ts
// scripts/check-db.ts

import { db } from '../src/lib/config/db/db';

async function checkDatabase() {
    try {
        console.log('🔍 Checking database connection...');
        console.log('DATABASE_URL:', process.env.DATABASE_URL?.substring(0, 50) + '...');
        console.log("Process env:", process.env);
        
        // 基本的な接続テスト
        const result = await db.execute('SELECT version()');
        console.log('✅ Database connection successful');
        
        // テーブル存在確認
        const tables = await db.execute(`
            SELECT table_name 
            FROM information_schema.tables 
            WHERE table_schema = 'public'
            ORDER BY table_name
        `);
        
        console.log('📋 Existing tables:');
        if (tables.rows.length === 0) {
            console.log('  No tables found - this is normal for initial deployment');
        } else {
            tables.rows.forEach((row: any) => {
                console.log(`  - ${row.table_name}`);
            });
        }
        
        console.log('✅ Database check completed successfully');
        process.exit(0);
        
    } catch (error) {
        console.error('❌ Database connection failed:', error);
        console.error('Error details:', {
            name: (error as Error).name,
            message: (error as Error).message,
        });
        process.exit(1);
    }
}

checkDatabase();
```
```sh
#!/bin/bash

echo "🚀 Starting Railway deployment process..."

# 環境変数の確認
if [ -z "$DATABASE_URL" ]; then
    echo "❌ DATABASE_URL is not set"
    exit 1
fi

echo "✅ Environment variables validated"

# データベースのスキーマを同期（テーブル作成）
echo "📊 Pushing database schema..."
npx drizzle-kit push --force --verbose

# データベースの初期化とシード
echo "🌱 Running database seed..."
npx tsx src/lib/config/db/init_db.ts

# アプリケーション起動
echo "🎯 Starting Next.js application..."
npm start
```

これをリモートリポジトリにPushすることで、自動的に`railway.toml`が読み込まれ、二つのスクリプトが実行されます。
Drizzle-ORMによる新規DBテーブルの自動作成や、既存テーブルへの新規要素追加などは`npx drizzle-kit push --force --verbose
`で行われます。
:::message alert
`--force`オプションは破壊的変更であっても強制Pushを行うので、drizzle-kitの最新バージョンで`--yes`オプションなどが追加された場合、速やかにそちらを使うことを推奨します。
:::


## Drizzle-ORMを使ったPostgresへのテーブル初期化
`drizzle.config.ts`と`データベース操作オブジェクトを定義しているファイル`について、以下のように記述してください。
```tsx
import { defineConfig } from "drizzle-kit";

// 環境変数の確認とログ出力
const databaseUrl = process.env.DATABASE_URL;
console.log("DATABASE_URL exists:", !!databaseUrl);
if (!databaseUrl) {
    throw new Error("DATABASE_URL environment variable is required");
}

export default defineConfig({
    schema: "./src/lib/config/db/schema/*.ts",
    out: "./drizzle",
    dialect: "postgresql",
    dbCredentials: {
        url: databaseUrl,
    },
    verbose: true,
    strict: true,
});
```

```tsx
// src/lib/config/db/db.ts

import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";

const connectionString = process.env.DATABASE_URL;

const pool = new Pool({
    connectionString: connectionString,
});

export const db = drizzle(pool);
```
そして、Railwayのリポジトリの`Variables`タブを開き、右上の`+New Variable`をクリックします。
![](/images/railway_one_practice/repo_variablesTab.png)
開かれた追加フォームから、`Add Reference`ボタンをクリックし、`DATABASE_URL`を選択し、`Add`ボタンをクリックしてください。
するとUI上でリポジトリのビルドとPostgresが矢印で接続され、「Railwayネットワーク内でアクセスが可能な状態になったこと」が示されます。
![](/images/railway_one_practice/connectByArrow.png)
> もしプロジェクト内で`DATABASE_URL`以外にも、`JWT_SECRET`などを読み取って処理を行なっている場合、それらの環境変数も全てこのページから追加してください。

ここまでの設定をリモートリポジトリにPushしてビルドが成功し、Postgresサービスの`Database`タブで以下のようにテーブルが作成されていれば成功です。
![](/images/railway_one_practice/onRailway_showPGTables.png)

# プロジェクトの外部公開
リポジトリのビルドの`Settings`タブを開き、`Networking`のところで`Generate Domain`というボタンを見つけてください。
![](/images/railway_one_practice/repo_changeNetGenDomain.png)

これをクリックすると、ドメインが自動生成され、誰でもアクセス可能なWebページとしてサービスが外部に公開されます。
![](/images/railway_one_practice/repo_generateDomain.png)


# 終わりに
ここまでのことが実行され、ビルドが成功すれば、Railwayネットワーク上でのバックエンドとフロントエンドの接続および変更の自動適用設定は完了です。
あとはリモートリポジトリに変更がある度に自動デプロイが行われ、公開されているWebページに変更が適用されます。
何か間違った解説箇所や、また、「自分はこうしたよ！」みたいな経験談があれば是非コメント等で教えていただけると嬉しいです。