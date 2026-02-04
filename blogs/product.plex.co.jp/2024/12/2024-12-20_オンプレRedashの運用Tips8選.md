---
title: オンプレRedashの運用Tips8選
published: 2024-12-20
updated: 2024-12-20T07:00:00+09:00
url: https://product.plex.co.jp/entry/maintenance-redash
entry-id: tag:blog.hatena.ne.jp,2013:blog-plexjp-26006613789038539-6802418398312515787
author: ishitsukajun
edited: 2026-01-07T17:28:27+09:00
tags:
  - Redash
  - データ基盤
  - コーポレートエンジニア
---

[f:id:plexjp:20260107172802p:plain]

こんにちは、プレックスの石塚です。この記事は、 [PLEX Advent Calendar 2024](https://qiita.com/advent-calendar/2024/plex)の20日目の記事です。

プレックスではBIツールとして2年ほど前からRedashを使用しています。今回の記事ではRedashの簡単な紹介とオンプレで運用する上でのTipsを8つ紹介します。

以前はMetabaseというBIツールを使用していたのですが、データ活用が思ったより進まず、Redashに移行する流れとなりました。このあたりの話もいずれ機会があればブログにまとめたいと思います。

[:contents]

# Redashとは？

[Redash](https://redash.io/)はPython製のオープンソースとして作られたBIツールです。自分が個人的に感じるRedashの最大の特徴はエンジニアフレンドリーであることです。下記の画像のように直接SQLを書いて、それを即座にグラフの形にビジュアライズできるため、エンジニアにとって直感的に使えるBIツールとなっています（APIの利用も非常に簡単です）。

<figure class="figure-image figure-image-fotolife" title="redash-introduction">[f:id:ishitsukajun:20241217094404p:plain]<figcaption></figcaption></figure>

そんな便利なRedashですが、2020年にDatabricksに買収されたことをきっかけに、クラウド版のサービス終了とOSS開発の停止という状況になってしまいました。そのため、現時点でRedashを使うためには、自社でサーバーをホスティングしてオンプレの環境で動かすことが必要です。

[https://redash.io/help/faq/eol/:embed:cite]

※2023年4月にコミュニティ主導のプロジェクトとして、OSS開発を再開するというアナウンスがありました。まだメジャーバージョンのリリースはありませんが、コントリビューションを見る限り着々と開発が進んでいるようです。

[https://github.com/getredash/redash/discussions/5962:embed:cite]

# オンプレRedashの運用Tips8選
以前はクラウド版があったため、運用について考える部分が少なかったのですが、今はオンプレで動かすしか選択肢がないということで、運用を楽にするためのTipsをいくつか紹介していきます。Redashのバージョンは `10.1.0` を想定しています。

## 1. パスワードログインを無効にする
Redashではパスワードログイン、Googleログイン、SAMLの3つの認証方法をサポートしています。パスワードログインを有効にしてしまうと、Redash側で退職者の整理などのアカウント管理をしなければならなくなるため、運用のコストが増えてしまいます。

[https://redash.io/help/user-guide/users/authentication-options/]

ログイン方法はRedashの画面上（ `/settings/general` ）から設定が可能です。

[f:id:ishitsukajun:20241217110255p:plain]

## 2. グループを使ったデータソースの管理
Redashでは権限をグループ×データソース×アクション（ `Full Access` or `View Only` ）で管理できます。複数の事業部やチームからの利用を想定している場合は、初期からグループの設計をしておくとよいでしょう。

[https://redash.io/help/user-guide/users/permissions-groups/]

## 3. redashbotを導入する
redashbotは、Slack上でRedashのURLをメンションするとグラフを展開してくれるツールです。Slack標準のリマインダーを使って、定期でグラフを流すといったことも可能です。

[https://raw.githubusercontent.com/yamitzky/redashbot/refs/heads/main/images/screenshot.png:image=https://raw.githubusercontent.com/yamitzky/redashbot/refs/heads/main/images/screenshot.png]

以前はRedashと同じインスタンス内で `docker run` で動かしていたのですが、たまにプロセスが落ちることがあったため、Redashを動かしている `docker-compose.yml` に `restart: always` オプションを付けて組み込んだところ、落ちなくなりました。

[https://github.com/yamitzky/redashbot:embed:cite]

## 4. クエリの同時実行数を調整する
Redashの手動でのクエリの同時実行数はデフォルトで2になっています。1つの事業部で使用している場合は問題にならないかもしれませんが、複数の事業部で複数のデータソースに対してクエリを実行したいケースでは、クエリの同時実行数を上げたい場合があります。

これは `docker-compose.yml` から `adhoc_worker` の環境変数である `WORKERS_COUNT` の数字で指定することで調整可能です。

```yml
  adhoc_worker:
    <<: *redash-service
    command: worker
    environment:
      QUEUES: "queries"
      WORKERS_COUNT: 4
```

## 5. Redashサイトの死活監視
オンプレで運用しているRedash上で、LIMITを付け忘れるなど取得するデータ量の多いクエリを投げてしまうと、サービスが落ちてしまうことが多々あります。プレックスではこれをすぐに検知できるように、Google Cloud MonitoringのUptime checksを使用して、Slackに通知しています。設定方法は下記のドキュメントを参照していただきたいのですが、簡単に死活監視を実現できます。

[https://cloud.google.com/monitoring/uptime-checks?hl=ja:title]

死活監視の上で工夫しているポイントとしては、通知先をRedashの利用者が集まるSlackのチャンネルにしていることです。以前は開発者しかいないエラー通知部屋に通知していたのですが、利用するメンバーにもサービスが落ちていることが周知できた方が良い、落ちる原因がほとんどクエリ起因なのでクエリ実行者に自覚してもらいたい、ということで変更しました。

また通知のメッセージには直近のクエリの実行時間、結果行数を調査するクエリのURLを付与しており、原因のクエリの調査をしやすくしています。Redash内部のPostgreSQLに対して発行できるクエリなので、参考までに貼っておきます。

```sql
SELECT
    '<a target="_blank" href="https://{domain}/queries/' || q.id || '">' || q.id || '</a>' AS id,
    q.name,
    q.user_id,
    u.name AS user_name,
    q.is_archived,
    q.is_draft,
    q.tags,
    qr.id AS query_result_id,
    qr.runtime AS seconds,
    JSON_ARRAY_LENGTH(qr.data::json->'rows') AS rows,
    qr.retrieved_at + INTERVAL '9 hours' AS retrieved_at_jst
FROM
    query_results AS qr
JOIN
    queries AS q
ON
    q.query_hash = qr.query_hash
JOIN
    users AS u
ON
    u.id = q.user_id
WHERE
    qr.retrieved_at > NOW() - INTERVAL '1 hour'
ORDER BY retrieved_at_jst
LIMIT 1000;
```

## 6. 定期実行の死活監視
Redashには[定期実行](https://redash.io/help/user-guide/querying/scheduling-a-query/)の機能がサポートされているのですが、プレックスで運用していたところ、定期実行が実行されていないというケースが稀にありました。そのため、サイト以外にも定期実行の死活監視も行っています。

方法としては、Railsのrakeタスクで定期実行をセットしたクエリの `/api/queries/<id>/results` APIを叩いて、クエリの最終実行時間（retrieved_at）がセットした時間内に行われているかをチェックしています。

[https://redash.io/help/user-guide/integrations-and-api/api/]

余談ですが、定期実行が失敗する根本の原因はRQ Schedulerのバージョンが古いことにあるようでした。コンテナの中に入って直接RQ Schedulerをバージョンアップしたところ、この問題は解決することができました。

[https://github.com/getredash/redash/issues/5797]
[https://dev.classmethod.jp/articles/fix-redash-scheduler-error-by-rq-scheduler/]

## 7. クエリのアーカイブ
Redashはクエリの検索機能の精度があまり良くなかったり、クエリを整理するためのフォルダ機能などがないためかクエリが乱立して探しづらいといった特徴があります。プレックスも例に漏れず、運用を開始して2年弱で約1,000個のクエリが作成されており、クエリを探すコストはどんどん上がっています。

命名やタグ付けのルールも定めていますが、いまいち浸透しきっていない状況です。そのため、2週間に1度、期間中に作成されたクエリを全部チェックして不要なクエリがあればアーカイブするという力技の作業を行っています。

UnpublishedなクエリのアーカイブなどはRedashの内部APIを使用すれば自動化できそうなので、部分的にやっていきたいと思っています。

## 8. いざという時のためのトラブルシューティング
実はRedashには内部のクエリの実行状況やキューの状況を確認できる管理画面が存在します。 `/admin/status` のURLから管理者権限を持っているユーザーのみアクセスできます。

[f:id:ishitsukajun:20241218103500p:plain]

クエリが実行中のまま終わらなくなってしまった、キューにクエリが溜まりすぎていて新しいクエリが実行できない等トラブルが発生した際の状況確認に上の管理画面は有用です。一方で管理画面から実行中のクエリを停止させる、キューの中身をクリアするといった機能は提供されていないため、Redashの内部APIを叩くかサーバーに入って作業する必要があります。

内部APIを叩く一例として、実行中のタスクを取得して、それを停止させるサンプルを載せておきます。

```zsh
% curl -s "https://{domain}/api/admin/queries/rq_status?api_key={api_key}" | jq .queues.queries
{
  "name": "queries",
  "started": [
    {
      "id": "e891f8ff-10b4-4b6f-a6c6-982cda1efa6f",
      "name": "redash.tasks.queries.execution.execute_query",
      "origin": "queries",
      "enqueued_at": "2024-12-18T01:51:43.744",
      "started_at": "2024-12-18T01:51:43.764",
      "meta": {
        "data_source_id": 6,
        "org_id": 1,
        "scheduled": false,
        "query_id": "adhoc",
        "user_id": 1
      }
    }
  ],
  "queued": 0
}
% curl -X DELETE "https://{domain}/api/jobs/e891f8ff-10b4-4b6f-a6c6-982cda1efa6f?api_key={api_key}"
null%
```

このようにRedashは内部で使用しているAPIをAPIキー（ `/users/me` から生成可能）があれば叩けるようになっています。もちろんドキュメントは用意されていないので、Githubのコードを直接読んで、自己責任での実行をお願いします。キューの中身を削除する場合は直接Redisに入ってredis-cliからクリアするといったことも可能です。

# おわりに
いくつかオンプレでRedashを運用するためのTipsをご紹介させていただきました。少しでも自社でRedashを運用している方の参考になれば幸いです。

コミュニティ主導のプロジェクトとして再稼働したRedashですが、これからの開発や新しいバージョンのリリースにも期待したいですね。
