他の提出者の方や、予選に参加されていた方の解き方をかなり参考にしました。

# 初期スコア
success:7960 fail:0 score:1720

# 最終スコア
workload 1の時
sccess:48190 fail:0 score:10410

workload 16の時
success:132690 fail:0 score:28668

workload 32の時
success: 132900 fail:0 score:28724

failがでない最大のworkloadの値が32であった。

# 変更した点
## nginxの設定変更
cat /proc/cpuinfo|grep processorするとプロセッサが4つなので，nginxのworker_processを4に変更した。

```
worker_processes  4;
```

## パスワードハッシュに関して比較を==から===に変更
パスワードハッシュの比較の==を===に置き換えた。 ==による緩やかな比較はセキュリティ上良い習慣とは言えず、 またキャストが発生するためパフォーマンス的にも===による厳密な比較には劣る。


## 不要なDBアクセスの削除
last_login()内でDBアクセスを含む関数current_user()を呼び出していた。
しかし返り値のうち使用しているのは$user_idのみであった。
これは$_SESSION['user_id']で事足りるため削除した。

## if文の並び替え
attempt_login()内のif-else文が!empty($user)を複数回実行する形になっていたので並び替えを行った。

## PHPの設定
php.fpmを以下のように書き換えた。

```
pm = static;
pm.max_children = 16;
pm_max_requests = 1024;
```


## Linuxの設定
使用するポートの数が多く、すぐ使いきってしまうため、再利用するよう/etc/sysctl.confを変更した。 
以下に変更した設定を記す。

```
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_rfc1337 = 1
net.ipv4.tcp_fin_timeout = 16
net.ipv4.tcp_keepalive_probes = 5
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 65535
```

## DBがメモリを使うように
使用しているEC2のインスタンスがxlargeと余裕があったため、メモリを使用するような設定に変更した。
以下に変更した設定を記す。 /etc/my.cnf

```
max_allowed_packet=3000M
max_connections=256
thread_cache=100
innodb_additional_mem_pool_size = 32M
innodb_buffer_pool_size = 8G
innodb_log_buffer_size = 64MB
join_buffer_size = 256M
read_buffer_size = 256M
read_rnd_buffer_size = 256M
sort_buffer_size = 256M
query_cache_size = 0
skip-innodb_doublewrite
skip-innodb_checksums
skip_name_resolve
```

## DBでインデックスの作成
init.shの末尾に以下の2行を追加。

```
mysql -h ${myhost} -P ${myport} -u ${myuser} ${mydb} -e 'create index user_id on login_log(user_id);'
mysql -h ${myhost} -P ${myport} -u ${myuser} ${mydb} -e 'create index ip on login_log(ip);'
```

これが一番効果が大きかった。