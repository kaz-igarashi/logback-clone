# RollingFileAppenderの働き
## ログ出力が行われた時

[Logback公式のアーキテクチャ](http://logback.qos.ch/manual/architecture_ja.html)にあるように、Appender#doAppend()が呼び出される。
Appender#doAppend()が呼び出された後の処理は以下のようになっている。

    UnsyncronizedAppenderBase#doAppend()
      OutputStreamAppender#append()
        RollingFileAppender#subAppend()
          triggeringPolicy.isTriggeringEvent()

RollingFileAppender.triggeringPolicyは、設定ファイルで`triggeringPolicyプロパティ`に設定したクラスが利用される。

## TimeBasedRollingPolicyの働き

- TimeBaseRollingPolicy#isTriggeringEvent()
    - DefaultTimeBasedFileNamingAndTriggeringPolicy#isTriggeringEvent() に処理を移譲
- DefaultTimeBasedFileNamingAndTriggeringPolicy#isTriggeringEvent()
    - `FileNamePatternプロパティ` に応じた「次のログ切り替え時刻」（TimeBasedFileNamingAndTriggeringPolicyBase#nextCheck）を過ぎていたら、ログの切り替え（rollover）を実行するため、trueを返す

## ログの切り替え

トリガになるのは、`RollingFileAppender#rollover()`

    RollingFileAppender#rollover()
      this.attemptRollover();
      this.attemptOpenFile();
    
    RollingFileAppdender#attemptRollover()
      rollingPolicy.rollover();
      
    TimeBaseRollingPolicy#rollover()
      ログファイルの圧縮など
      archiveRemover.cleanAsynchronously(now); # 古いファイルの削除

## TimeBaseRollingPolicy#archiveRemover
- `TimeBaseRollingPolicy#start()`で、`TimeBasedFileNamingAndTriggeringPolicyBase.getArchiveRemover()` を呼び出して設定している
    - TimeBaseFileNamingAndTriggeringPolicyBase#getArchiveRemover()で返ってくるのは、`TimeBasedArchiveRemover`になる

# TimeBasedArchiveRemoverでのログ削除

## TimeBaseArchiveRemover#cleanAsynchronously()
別スレッドで削除を行うため、ExecuterServiceに`ArhiveRemoverRunnable`のインスタンスをsubmitする。

## ArchiveRemoverRunnable
- run()
    - TimeBaseArchiveRemover#clean() の呼び出し
    - TimeBaseArchiveRemover#capTotalSize() の呼び出し

## TimeBaseArchiveRemover#clean()
    TimeBaseArchiveRemover#clean()
      periodsElapsed = TimeBaseArchiveRemover#computeElapsedPeriodsSinceLastClean()
        前回の削除実施時刻`lastHeartBeat` からの経過ミリ秒 ÷ rollover単位のミリ秒（1日単位なら、24*60*60*1000=86400000）
        前回の削除実施が行われていない場合は、「前回の実施時刻を32日前」とした経過ミリ秒 ÷ rollover単位のミリ秒
      for(int i=0; i < periodsElapsed; i++) {
        削除時刻 = 現在時刻から(-maxHistory - 1 - i)*rollover単位
        TimeBaseArchiveRemover#cleanPeriod(削除時刻)
      }
      
## TimeBaseArchiveRemover#cleanPeriod()
- 指定された時刻を、`fileNamePatternプロパティ`に適用して、削除対象とするファイル名を生成する
- 生成したファイル名が、ファイルでかつ存在していた場合は、削除する


## ArchiveRemoverRunnableが実行されるタイミング
ExecuterServiceはJava標準の`java.util.concurrent.ScheduledThreadPoolExecutor`を使用している。
→遅延時間の指定をせずにsubmit()しているため、基本的には即時実行される。

## cleanHistoryOnStartの設定
`cleanHistoryOnStartプロパティ`が`true`の場合、`TimeBasedRollingPolicy.start()`で`TimeBaseArchiveRemover#cleanAsynchronously()`を呼び出している。


# まとめ
- ログのローテーションは、「ログファイルへの書き込みが必要なログ出力メソッドが呼び出された時刻」を元に判定される
- 過去のログ削除は、ログ出力とは別スレッドで実施される
  - 別スレッドは、基本的に即時に起動されるが、別のログ削除のスレッドが動作している場合は待たされることになる
- 削除対象のファイル名は、（ログの出力時刻 - 32日）が最大となっている
  - これは、`cleanHistoryOnStart`が`true`になっていても同じ処理が行われる
  