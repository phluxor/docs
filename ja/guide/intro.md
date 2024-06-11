# はじめに

このガイドでは、Phluxorの使い方の前に簡単なアクターモデルの説明を行います。  

PHPの多くの伝統的なプログラミングモデルでは、  
プログラムはひとつのスレッドで実行され同期的に処理が行われます。  

リクエスト・レスポンスモデルやイベント駆動モデルなど、  
多くのプログラミングモデルは、この同期的な処理を前提としています。  

しかし、近年のWebアプリケーションや分散システムでは、  
複数のリクエストやイベントを同時に処理する必要があります。  
クラウドコンピューティングやマイクロサービスアーキテクチャなど、  
時代の変化に合わせて、アプリケーションの処理モデルも変化しています。  

このような状況に対応するために、アクターモデルが注目されています。  
アクターモデルは、並行処理を行うためのモデルのひとつです。  

アクターモデルは、メッセージを送信することで他のアクターに通信を行い、  
メッセージを受信することで他のアクターから通信を受け取ります。  

アクターモデルは、アクターという単位で処理を行い、  
それぞれのアクターは独立して動作します。  
ゆえに負荷分散やスケーラビリティといった要件に対応しやすい特徴があります。  

これまでのPHPアプリケーションではそうした対応は難しかったですが、  
Phluxorを使うことでアクターモデルをPHPで簡単に実現し、  
アプリケーションの処理モデルを柔軟に変更できるようになります。  

Phluxorでは多くのアクターモデルツールキットと同様に、  
アクターの作成やメッセージの送受信、アクターの監視などの機能を提供します。  

「let it crash」の哲学に基づいて、アクターはエラーを自己処理し、  
他のアクターに影響を与えないように設計されています。  

[Akka](https://akka.io/)や[Pekko](https://pekko.apache.org/)、  
[Proto Actor](https://proto.actor/)などのアクターモデルツールキットと同様の概念に基づいていますので、  
これらのツールキットを使ったことがある方はすぐに使い方を理解できるでしょう。

逆にそうしたツールキットを使ったことがない方はPhuxorを使ってアクターモデルを学ぶことで、  
他のツールキットを使ったときにも応用できる知識を身につけることができます。  

![Actors](/images/actors.png "Actors")

アクターはヒエラルキー構造を持ち、親アクターから子アクターへメッセージを送信できます。  
すべてのアクターは、ルートアクターを親アクターとして持ち、  
メッセージを送信することで他のアクターに通信を行います。  
このアクターはすべてアドレスを持ち、アドレスを使って他のアクターにメッセージを送信します。  

PHPをはじめとする多くの言語でおなじみの `return` は使用しません。  
これまでのプログラミングパラダイムとは異なる、  
アクターモデルの新しい世界を体験してみましょう。  

[インストール >](/ja/guide/install.html)