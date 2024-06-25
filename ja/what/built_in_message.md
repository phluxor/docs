# Built In Messages

Phluxor内部で利用されているメッセージの一覧です。  

## Supervision

### Phluxor\ActorSystem\Message\Failure

アクターが例外をスローしたときに送信されるメッセージです。  
アクターはこのメッセージを受信してエラー処理を行います。  

このとき該当アクターのMailboxはSuspended状態になり、  
メッセージの処理が一時停止されます。  

### Phluxor\ActorSystem\Message\ResumeMailbox

アクターのMailboxがSuspended状態になっているときに送信されるメッセージです。  
アクターはこのメッセージを受信してMailboxの処理を再開します。  
アクター再起動時にも内部的に送信されます。  

### Phluxor\ActorSystem\Message\SuspendMailbox

アクターはこのメッセージを受信してMailboxの処理を一時停止します。  
`Phluxor\ActorSystem\Message\Failure` メッセージを受信したときに内部的に送信され、  
MailboxがSuspended状態になります。  

### Phluxor\ActorSystem\ProtoBuf\Watch

アクターが他のアクターを監視するためのメッセージです。  
Futureを利用する際に内部的に送信されます。  
このメッセージを受信したアクターは、監視対象アクターの状態を監視し、  
監視対象アクターが終了したときに `Phluxor\ActorSystem\ProtoBuf\Terminated` メッセージを受信します。  

ルーターやスーパーバイザーなど、アクターの監視を行うアクターが利用します。  
任意で利用することもできます。  

### Phluxor\ActorSystem\ProtoBuf\Unwatch

アクターが他のアクターの監視を解除するためのメッセージです。  
`Phluxor\ActorSystem\ProtoBuf\Watch` メッセージを受信したアクターが、  
監視対象アクターの監視を解除するときに内部的に送信されます。  

## Lifecycle

### Phluxor\ActorSystem\ProtoBuf\PoisonPill

アクターを終了し、メッセージ キューを停止するために使用されます。  
アクターのMailboxにあるメッセージが処理されたあとにアクターを終了させます。  

SystemMessageではなく、通常のメッセージとして送信されるため  
受信順に処理されます。  

### Phluxor\ActorSystem\Message\Restart

アクターを再起動するためのメッセージです。  

### Phluxor\ActorSystem\Message\Restarting

アクター再起動中に送信されるメッセージです。  

### Phluxor\ActorSystem\Message\Started

アクターが開始されたときに送信されるメッセージです。  
アクターをSpawnしたときに内部的に送信されます。  

### Phluxor\ActorSystem\ProtoBuf\Stop

Mailboxのユーザーメッセージ有無を問わず、  
直ちにアクターを直ちに停止させるために利用します。  

### Phluxor\ActorSystem\Message\Stopping

アクター停止中を表すメッセージです。  

### Phluxor\ActorSystem\Message\Stopped

アクターが停止されたときに送信されるメッセージです。  

### Phluxor\ActorSystem\ProtoBuf\Terminated

監視対象アクターが終了したときに送信されるメッセージです。  
