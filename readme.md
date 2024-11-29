# 最初のスコア
```json
{"pass":true,"score":477,"success":659,"fail":12,"messages":["リクエストがタイムアウトしました (GET /)","リクエストがタイムアウトしました (GET /logout)","リクエストがタイムアウトしました (POST /login)","リクエストがタイムアウトしました (POST /register)"]}
```

# 修正後のスコア
```json
{"pass":true,"score":7682,"success":6915,"fail":0,"messages":[]}
```

# やったこと
最初のrunのとき mysql container が 100% になったことを気づき、app container が 30% ほどなので、ボトルネックがmysqlにあるとわかる。

```sql
SELECT 
    OBJECT_SCHEMA,
    OBJECT_NAME,
    COUNT_READ,
    COUNT_WRITE,
    COUNT_FETCH
FROM 
    performance_schema.table_io_waits_summary_by_table
WHERE 
    OBJECT_SCHEMA NOT IN ('mysql', 'performance_schema', 'information_schema')
ORDER BY 
    (COUNT_READ + COUNT_WRITE + COUNT_FETCH) DESC
LIMIT 10;
```
を実行して、以下の出力を得た
+---------------+-------------+------------+-------------+-------------+
| OBJECT_SCHEMA | OBJECT_NAME | COUNT_READ | COUNT_WRITE | COUNT_FETCH |
+---------------+-------------+------------+-------------+-------------+
| isuconp       | comments    |  250605368 |      100004 |   250605368 |
| isuconp       | posts       |     730576 |       10003 |      730576 |
| isuconp       | users       |       6930 |        1045 |        6930 |
+---------------+-------------+------------+-------------+-------------+

comments が大量に read されていることがわかる。
ここで comments table の構造を見てみると

```sql
CREATE TABLE `comments` (
  `id` int NOT NULL AUTO_INCREMENT,
  `post_id` int NOT NULL,
  `user_id` int NOT NULL,
  `comment` text NOT NULL,
  `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=100005 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci |
```

indexがないことがわかる。posts, users, commentsの構造なので、大量にpost_idとuser_idからのクエリーがあると想像がつく。
具体的にはSlow Query LogやGeneral Query Logなどを起動して、どのcolumからアクセスしているかを見るべきですが、ここで一旦 post_id だけ index にすることにしてみた
```sql
ALTER TABLE comments ADD INDEX post_id_idx (post_id);
```

ここで再度実行benchmarkを実行。以下の出力を得た。
```json
{"pass":true,"score":7682,"success":6915,"fail":0,"messages":[]}
```

スコアは上がって、mysql の負荷も40%ほどまで下がった。めでたし。
しかし app container の負荷が 100% になったので、ボトルネックが app（今回はphp起動）にあるとわかる。

これからコードに入って、コードレベルのチューニングを行っていくのですが、phpはあまり触ったことないので、今回はここまでにします。
