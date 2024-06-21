# What is Phluxor?

Phluxorは、PHPでアクターモデルを実装するためのライブラリです。  
アクターモデルは、並行処理を行うためのモデルのひとつで、  
アクターという単位で処理を行います。  

## Relation To Akka/Pekko and Proto Actor

PhluxorはAkkaやPekko、Proto Actorなどのアクターモデルツールキットと同様の概念に基づいています。  

すべてのアクターは、ルートアクターを親アクターとして持ち、  
ヒエラルキー構造を持ち、親アクターから子アクターへメッセージを送信できます。  
またこれらは完全に独立して動作し、他のアクターに影響を与えません。  

Swoole/OpenSwooleを使用して非同期処理を行いますが、  
保守運用が容易で、高速な処理が可能です。  
実装時には、SwooleのAPIを直接使用することはなく、  
PhluxorのAPIを使用してアクターモデルを実装します。  
並行処理についての知識がなくても、簡単にアクターモデルを実装できるように設計されています。  

アクター間の通信はProtocol Buffersを使用していますが、  
メッセージの永続化を利用しない場合は、  
通常のPHPのオブジェクトやプリミティブ型を使用することもできます。  

今後gRPCなどにも対応する予定です。  

## Scalable, distributed systems

PHPのアプリケーションは、従来の同期的な処理を前提としているため、  
複数のリクエストやイベントを同時に処理することが難しいです。  

Phluxorを使うことで、アクターモデルをPHPで簡単に実現し、  
新しいプログラミングパラダイムを導入することで、  
アプリケーションの処理モデルを柔軟に変更できるようになります。  

基本的にはクラッシュしても問題ないように設計されており、  
アクターはエラーを自己処理し、他のアクターに影響を与えないように設計されています。  

負荷分散やスケーラビリティといった要件に対応しやすい特徴があり、  
クラウドコンピューティングやマイクロサービスアーキテクチャなど、  
時代の変化に合わせたあたらしいPHPアプリケーションを作成できます。  

Phluxorは、WebアプリケーションやCLIアプリケーションなど、  
さまざまなアプリケーション・フレームワークで利用できます。  
Webアプリケーションとしての機能は持っていませんので、  
Webアプリケーションを作成する場合は、  
SwooleのHTTPサーバー等を使ってWebアプリケーションを作成してください。