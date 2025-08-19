Spring Batch の処理シーケンス図を「ジョブ実行」を軸にして整理しました。
一般的な JobLauncher → Job → Step → ItemReader / ItemProcessor / ItemWriter の流れを表現します。


---

Spring Batch シーケンス図（概念レベル）

sequenceDiagram
    participant Client as クライアント
    participant JobLauncher as JobLauncher
    participant Job as Job
    participant Step as Step
    participant Reader as ItemReader
    participant Processor as ItemProcessor
    participant Writer as ItemWriter
    participant Repository as JobRepository

    Client->>JobLauncher: ジョブ起動 (JobParameters)
    JobLauncher->>Repository: ジョブ実行情報を登録
    JobLauncher->>Job: ジョブ開始
    
    loop 各Step
        Job->>Step: Step開始
        Step->>Repository: Step実行情報を登録
        
        loop Chunkごとの処理
            Step->>Reader: データ読み込み
            Reader-->>Step: Itemデータ
            
            Step->>Processor: データ加工/検証
            Processor-->>Step: 加工済みデータ
            
            Step->>Writer: データ書き込み
            Writer-->>Step: 書き込み結果
        end
        
        Step->>Repository: Step実行結果を更新
    end
    
    Job->>Repository: ジョブ実行結果を更新
    Job-->>JobLauncher: ジョブ完了
    JobLauncher-->>Client: 実行結果返却


---

ポイント解説

1. JobLauncher

ジョブ実行のエントリポイント。

JobRepository に実行開始を記録した上で Job を起動します。



2. Job

複数の Step で構成される。

Step の実行順序や条件分岐を制御可能。



3. Step

チャンク指向処理が一般的。

ItemReader → ItemProcessor → ItemWriter の流れで 1チャンク単位の処理を繰り返す。



4. JobRepository

ジョブ/ステップの開始・進行・終了状態を永続化し、再実行やリスタートを可能にする。




