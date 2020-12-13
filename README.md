# Log Test for kubernetes
ログの出力がfluentdで欠損なくできているかを確かめるためのスクリプト

## docker build
```
docker build -t ntsutake/logtest .
docker push ntsutake/logtest
```

## run with k8s
```
kubectl apply -f logtest.yaml
```

## edit fluentd
sbifi-fluentdのfluentd-stg.yamlにおいて以下を追記する。

```
  fluent.conf: |
     ....
+    @include fluentd.conf

  base.conf: |
     ...
+        <store>
+          @type relabel
+          @label @test
+        </store>

```

以下を追加する。

```
  test.conf: |
    <label @test>
      <filter **>
        @type record_modifier
        <record>
          pod_name ${record["kubernetes"]["pod_name"]}
          container_name ${record["kubernetes"]["container_name"]}
        </record>
      </filter>
      <filter **>
        @type record_modifier
        <record>
          group_name /aws/eks/stg-tf-ekscluster/containerlog/logtest
          stream_name ${record["kubernetes"]["pod_name"]}
          log_messages ${record["message"]}
          retention_day 7
        </record>
      </filter>
      <match **>
        @type relabel
        @label @cloudwatch_logs
      </match>
    </label>
```

## check with cloudwatch log insight
以下のクエリで10000レコードが検索されることを確認する

```
fields @timestamp, @message
| stats count(*)
```
