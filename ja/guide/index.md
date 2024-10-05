# Phluxor Guide

## Getting Started

- [はじめに](intro.md)
- [インストール](install.md)
- [Hello World](hello.md)

## Concepts

- [設計原則](/ja/what/principles.html)
- [組み込みメッセージについて](/ja/what/built_in_message.html)
- [アクターライフサイクル](/ja/what/lifecycle.html)
- [Supervision](/ja/what/supervision.html)

## Features

- [Actor](/ja/features/actor.html) - アクターとは？
- [Props](/ja/features/props.html) - アクターの設定方法って？
- [Spawning Actors](/ja/features/spawn_actors.html) - アクターの生成方法は？
- [Ref](/ja/features/ref.html) - リファレンスって何？あくたーのアドレス？
- Context - コンテキストって何？
- Mailbox - アクターはメッセージをどう処理する？
- DeadLetter - 失われたメッセージはどうなる？
- Timeout - メッセージの受信タイムアウトを設定する方法は？
- [Future](/ja/features/future.html) - アクターの処理結果を取得する方法は？
- [Behavior](/ja/features/behavior.html) - アクターの状態を変更する方法は？
- Typed Channel - 型付きチャネルって何？
- Middleware - ミドルウェアって何？
- Reentrancy - 再入処理をどう扱う？
- Router - メッセージをアクターにどうルーティングする？
    - RoundRobin - ラウンドロビンルーティング
    - Random - ランダムルーティング
    - Broadcast - ブロードキャストルーティング
    - ConsistentHash - 一貫性ハッシュルーティング
- EventStream - イベントの発行と購読方法は？
- Persistence - アクターの状態をどう永続化する？
    - MySQL Provider - MySQLでアクターの状態を永続化する方法は？
    - PostgreSQL Provider - PostgreSQLでアクターの状態を永続化する方法は？
    - DynamoDB Provider - DynamoDBでアクターの状態を永続化する方法は？
- Remoting - 他のアクターシステムのアクターとどう通信する？

## Phluxor Training

Soon.  

- サーガパターンの実装
- スキャッターアンドギャザーの実装
- CQRS/ESの実装
