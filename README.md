# Percona XtraDB Cluster

![ansible ci](https://github.com/link-u/ansible-roles-v2_percona-cluster/workflows/ansible%20ci/badge.svg)

## 概要
Percona XtraDB Cluster をインストールするための ansible role について各変数の意味や使い方を説明している.
また, bootstrap.yml の使い方については[別の README](README_others/README_bootstrap.md) として用意している.

<br>

## 動作確認バージョン

* Ubuntu: 18.04, 20.04
* ansible: 2.8, 2.9

<br>

## 使用ポート番号
| ポート　 | 説明 |
| :------- | :--- |
| 3306/tcp | 通常の MySQL ポート |
| 4567/{tcp,udp} | クラスタ間での通信ポート |
| 4444/tcp | スナップショット状態転送 (state snaphot transfer: SST) 用のポート |
| 4568/tcp | インクリメンタル状態転送 (incremental state transfer: IST) 用のポート |

<br>

## 使い方 (ansible)

### Role variables

```yaml
### インストール設定 ###############################################################################
## 基本設定
pxc_install_flag: True  # インストールフラグ
pxc_version: 57
pxc_package_name: "percona-release_latest.generic_all.deb"
pxc_mycnf_path: "/etc/mysql/my.cnf"
pxc_root_password: "test&root" # デフォルトでは "default&root&password" となっており, 変更しないとエラーを出して止める設計
pxc_pid_file: "/var/run/mysqld/mysqld.pid"
pxc_socket: "/var/run/mysqld/mysqld.sock"

## クラスタ設定
pxc_name: "link-u_pxc"
pxc_sst_user: "pxc_sst_user"
pxc_sst_password: "test&sst" # デフォルトでは "default&sst&password" となっており, 変更しないとエラーを出して止める設計
pxc_hosts: "{{ (group_names | map('extract', groups) | list)[0] }}"
pxc_host_ips: "{{ pxc_hosts | map('extract', hostvars, 'local_ipv4') | list }}"
pxc_is_clustered: "{{ (pxc_hosts | length) > 1 }}"

## DB 操作ノード
# デフォルでの指定ノードについて
#   1. クラスタ稼働の場合はプレイ対象リストにおける最初のホストを指定
#   2. スタンドアロン稼働の場合は ansible 実行対象のホストを指定
pxc_db_control_node: "{% if pxc_is_clustered %}{{ ansible_play_hosts_all[0] }}{% else %}{{ inventory_hostname }}{% endif %}"

## NUMA環境下のメモリ割り当て改善設定
### innodb_numa_interleaveとflush_cachesの設定値に使用する
pxc_enable_innodb_numa_interleave: "{{ (ansible_processor_count >= 2) | ternary('ON', 'OFF') }}"


### 追加設定 #######################################################################################
## 作成する mysql ユーザリスト
pxc_users: []
### 作成例
# pxc_users:
#   - name: "foo"           # ユーザ名 (必須)
#     password: "foofoo"    # パスワード (省略時は name と同じ値)
#     priv: "*.*:ALL,GRANT" # ユーザの権限 (省略時は '*.*:ALL,GRANT')
#     hosts:                # 接続できるホスト名リスト (省略はできるが, その場合ユーザ作成タスクは走らない)
#       - "127.0.0.1"
#       - "::1"
#       - "localhost"
#
#   - name: "bar"           # ユーザ名.  パスワードは省略
#     priv: "*.*:SELECT"    # ユーザの権限
#     hosts:                # 接続できるホスト名リスト
#       - "localhost"


## 作成する mysql DB リスト
pxc_databases: "{{ percona_cluster_databases | default([]) }}"
### 作成例
# pxc_databases:
#   - name: somedb                   # DB 名 (必須)
#     encoding: utf8mb4              # DB の文字コード (省略時は "utf8" )
#     collation: utf8mb4_general_ci  # 照合モード (省略時は "" ※ 空文字)
#     login_user: root               # DB 作成時のログインユーザ (省略時は root)
#     login_password: password       # DB 作成時のログインパスワード (省略時は "{{ pxc_root_password }}")
#
#   - name: testdb                   # DB 名以外省省略した場合


### cnf ファイル 設定 ##############################################################################
## clustering_settings
#   * 辞書のマージに対応
#   * 辞書変数の一部だけを修正可能.
pxc_clustering_settings:
  wsrep_provider: "{{ pxc_is_clustered | ternary('/usr/lib/libgalera_smm.so', 'none') }}"
  wsrep_cluster_address: "gcomm://{{ pxc_host_ips | join(',') }}"
  wsrep_cluster_name: "{{ pxc_name }}"
  wsrep_log_conflicts: 1
  wsrep_node_address: "{{ local_ipv4 }}"
  wsrep_node_name: "{{ inventory_hostname }}"
  wsrep_slave_threads: "{{ ((ansible_virtualization_role == 'host') | ternary(ansible_processor_count * ansible_processor_cores, ansible_processor_nproc))}}"  # 目安はCPUコアx2だがちょっと多いので減らす
  wsrep_sst_method: xtrabackup-v2
  wsrep_sst_auth: '"{{ pxc_sst_user }}:{{ pxc_sst_password }}"'
  wsrep_data_home_dir: /var/lib/mysql
  wsrep_provider_options: "'gcache.dir = /var/lib/mysql'"
  server-id: "{{ pxc_hosts.index(inventory_hostname) }}"

## mycnf_settings
#   * 辞書のマージに対応
#   * 辞書変数の一部だけを修正可能.
pxc_mycnf_settings:
  user: mysql
  port: 3306
  character-set-server: utf8mb4
  collation-server: utf8mb4_general_ci
  default-authentication-plugin: "mysql_native_password"
  plugin-load-add: "auth_socket.so"
  ignore-db-dir: ["mysql-bin", "lost+found"]  # ignore-db-dir の設定 (リストで指定)
  datadir: /var/lib/mysql
  log-bin: /var/lib/mysql/mysql-bin
  log-error: /var/log/mysql/error.log
  socket: "{{ pxc_socket }}"
  table_open_cache_instances: "{{ ((ansible_virtualization_role == 'host') | ternary(ansible_processor_count * ansible_processor_cores, ansible_processor_nproc)) * 2 }}"  # コア数×2
  thread_handling: one-thread-per-connection # perconaのウリである pool-of-threads を指定したいが、謎のメモリリーク問題により断念
  pid-file: "{{ pxc_pid_file }}"
  open_files_limit: 100000
  log_slave_updates: 1
  skip_character_set_client_handshake: 1
  skip_external_locking: 1
  skip_name_resolve: 1
  symbolic-links: 0
  performance_schema: "OFF"
  table_open_cache: "{{ ((ansible_virtualization_role == 'host') | ternary(ansible_processor_count * ansible_processor_cores, ansible_processor_nproc)) * 5000 }}"  # 目安はコア数x5000
  binlog_format: ROW
  default_storage_engine: InnoDB
  pxc_strict_mode: ENFORCING
  innodb_autoinc_lock_mode: 2
  innodb_buffer_pool_instances: "{{ [(((ansible_virtualization_role == 'host') | ternary(ansible_processor_count * ansible_processor_cores, ansible_processor_nproc)) * 2), 64] | min }}" # コア数の2倍か64の小さい方
  innodb-checksum-algorithm: strict_crc32
  innodb_flush_method: O_DIRECT_NO_FSYNC
  innodb_log_buffer_size: 32M  # 16-128MB
  innodb_log_file_size: 128M
  innodb_log_files_in_group: "{{ ((ansible_virtualization_role == 'host') | ternary(ansible_processor_count * ansible_processor_cores, ansible_processor_nproc)) }}"  # コア数
  innodb_log_group_home_dir: /var/lib/mysql
  innodb_numa_interleave: "{{ pxc_enable_innodb_numa_interleave }}"
  innodb_page_cleaners: "{{ [(((ansible_virtualization_role == 'host') | ternary(ansible_processor_count * ansible_processor_cores, ansible_processor_nproc))), 32] | min }}"    # コア数か32の小さい方
  innodb_purge_threads: "{{ [(((ansible_virtualization_role == 'host') | ternary(ansible_processor_count * ansible_processor_cores, ansible_processor_nproc))), 32] | min }}"    # コア数か32の小さい方
  innodb_read_io_threads: "{{ [(((ansible_virtualization_role == 'host') | ternary(ansible_processor_count * ansible_processor_cores, ansible_processor_nproc))), 32] | min }}"  # コア数か32の小さい方
  innodb_write_io_threads: "{{ [(((ansible_virtualization_role == 'host') | ternary(ansible_processor_count * ansible_processor_cores, ansible_processor_nproc))), 32] | min }}" # コア数か32の小さい方

## general_settings
#   * 辞書のマージに対応
#   * 辞書変数の一部だけを修正可能.
#   * SET GLOBALが可能なものは出来るだけこちらへ
#   * SET GLOBALは単位にMとか使えないので`max_heap_table_size`などを参考にして下さい。
pxc_general_settings:
  innodb_adaptive_hash_index: "OFF"
  innodb_flush_log_at_trx_commit: 0
  innodb_flush_neighbors: 0
  innodb_io_capacity: 2000  # NVMeでなければ2000ぐらいでよい
  innodb_io_capacity_max: 4000 # innodb_io_capacity の2倍
  innodb_lru_scan_depth: 1024  # NVMeでなけれ1024でよい
  #innodb_page_size: 4k  # clusterのバグによりデフォルトの16kしか指定できない
  innodb_random_read_ahead: "OFF"
  innodb_read_ahead_threshold: 0
  innodb_compression_level: 6
  innodb_parallel_doublewrite_path: xb_doublewrite # ダブルライトバッファのパスを変更する
  ## メモリの5割くらい、chunk_size(128MB) * instances の倍数でしか変更できない
  innodb_buffer_pool_size: "{{ ((((ansible_memtotal_mb * 1024 * 1024 * 0.5) / (128 * 1024 * 1024 * __pxc_combined_mycnf_settings.innodb_buffer_pool_instances|int))|round(0, 'ceil')) * 128 * 1024 * 1024 * __pxc_combined_mycnf_settings.innodb_buffer_pool_instances|int)|int }}"
  expire_logs_days: 3
  explicit_defaults_for_timestamp: "ON"
  long_query_time: 0.1
  log_timestamps: SYSTEM
  max_connections: "{{ ((ansible_virtualization_role == 'host') | ternary(ansible_processor_count * ansible_processor_cores, ansible_processor_nproc)) * 200 }}"  # 目安はコア数x200
  query_cache_limit: 0
  query_cache_size: 0
  slow_query_log: "ON"
  slow_query_log_file: /var/log/mysql/mysql-slow.log
  thread_cache_size: "{{ ((ansible_virtualization_role == 'host') | ternary(ansible_processor_count * ansible_processor_cores, ansible_processor_nproc)) * 100 }}"  # 目安はコア数x100
  default_password_lifetime: 0
  max_prepared_stmt_count: 1000000
  ## 以下は本番サーバー（メモリ16GB以上）と開発サーバー（メモリ16GB未満）で推奨値を分ける
  max_heap_table_size: "{% if ansible_memtotal_mb >= 16 * 1024 %}{{ 128 * 1024 * 1024 }}{% else %}{{ 16 * 1024 * 1024 }}{% endif %}" # 128MB or 16MB
  tmp_table_size: "{% if ansible_memtotal_mb >= 16 * 1024 %}{{ 128 * 1024 * 1024 }}{% else %}{{ 16 * 1024 * 1024 }}{% endif %}"      # 128MB or 16MB
  read_buffer_size: "{% if ansible_memtotal_mb >= 16 * 1024 %}{{ 4 * 1024 * 1024 }}{% else %}{{ 128 * 1024 }}{% endif %}"            # 4MB or 128KB
  read_rnd_buffer_size: "{% if ansible_memtotal_mb >= 16 * 1024 %}{{ 4 * 1024 * 1024 }}{% else %}{{ 256 * 1024 }}{% endif %}"        # 4MB or 256KB
  join_buffer_size: "{% if ansible_memtotal_mb >= 16 * 1024 %}{{ 4 * 1024 * 1024 }}{% else %}{{ 256 * 1024 }}{% endif %}"            # 4MB or 256KB
  sort_buffer_size: "{% if ansible_memtotal_mb >= 16 * 1024 %}{{ 4 * 1024 * 1024 }}{% else %}{{ 256 * 1024 }}{% endif %}"            # 4MB or 256KB
  sync_binlog: 0
```

<br>

### Example playbook

```yaml
- hosts:
    - servers
  become: True
  roles:
    - { role: percona-cluster, tags: ["percona-cluster"] }
```

<br>

## 各種変数について

### 1. 基本設定に関する変更可能な変数
| 変数　　 | デフォルト値   | コメント     | 変更の必要性 |
| :------- | :------------- | :----------- | :----------- |
| `pxc_install_flag` | `True` | `False` の時は[インストールタスク](./tasks/install.yml) は実行しない | インストールと設定用で playbook をかき分ける場合. |
| `pxc_version` | `57` | Percona XtraDB Cluster のバージョン | 一応, 明示的に変更しておいたほうがいい |
| `pxc_mycnf_path` | `/etc/mysql/my.cnf` | `my.cnf` のファイルパス | 変更の必要なし |
| `pxc_root_password` | `test&root` | mysql の `root` ユーザのパスワード | 必ず変えるべき!! |
| `pxc_pid_file` | `/var/run/mysqld/mysqld.pid` | mysql の pid ファイルのパス | 変更の必要なし |
| `pxc_socket` | `/var/run/mysqld/mysqld.sock` |  mysql の socket ファイルのパス | 変更の必要なし |
| `pxc_ignore_db_dir` | `["mysql-bin", "lost+found"]` | `my.cnf` における `ignore-db-dir` の指定 | 必要に応じて変更 |

<br>

### 2. クラスタ設定に関する変更可能な変数
| 変数　　 | デフォルト値   | コメント     | 変更の必要性 |
| :------- | :------------- | :----------- | :----------- |
| `pxc_name` | `link-u_pxc` | クラスタ名 | 変更の必要なし |
| `pxc_sst_user` | `pxc_sst_user` | クラスタ間の同期用ユーザ | 変更の必要なし |
| `pxc_sst_password` | ここでは秘密 (ソースを読むこと) | sst_user 用のパスワード | 必ず変えるべき!! |
| `pxc_is_clustered` | 対象ホストが2台以上:  `True`<br> 対象ホストが1台: `False` | クラスタモードかスタンドアロンモードかを決める. | 開発環境の場合は変える必要なし <br> 本番環境の場合は1台であっても, スケールアウトにも対応することを考えて `True` の方がいい.  |

<br>

### 3. DB へのユーザ追加用の変数と DB へのテーブル追加用の変数
| 変数　　 | デフォルト値   | コメント     | 変更の必要性 |
| :------- | :------------- | :----------- | :----------- |
| `pxc_user` | `[]` (空のリスト) | DB にあらかじめ用意するユーザ | プロジェクトに応じて追加 |
| `pxc_databases` | `[]` (空のリスト) | DB にあらかじめ用意するテーブル | プロジェクトに応じて追加 |

`pxc_user` 定義の仕方
```yaml
pxc_users:
  ## 基本的な書き方
  - name: "foo"           # ユーザ名 (必須)
    password: "foofoo"    # パスワード (省略時は name と同じ値)
    priv: "*.*:ALL,GRANT" # ユーザの権限 (省略時は '*.*:ALL,GRANT')
    hosts:                # 接続できるホスト名リスト (省略はできるが, ユーザ作成タスクは走らない)
      - "127.0.0.1"
      - "::1"
      - "localhost"

  ## 省略を利用した例
  - name: "bar"           # ユーザ名.  パスワードは省略
    priv: "*.*:SELECT"    # ユーザの権限
    hosts:                # 接続できるホスト名リスト
      - "localhost"
```

`pxc_user` 定義の仕方
```yaml
pxc_databases:
  ## 基本的な書き方
  - name: somedb                   # DB 名 (必須)
    encoding: utf8mb4              # DB の文字コード (省略時は "utf8" )
    collation: utf8mb4_general_ci  # 照合モード (省略時は "" ※ 空文字)
    login_user: root               # DB 作成時のログインユーザ (省略時は root)
    login_password: password       # DB 作成時のログインパスワード (省略時は "{{ pxc_root_password }}")

  ## 省略を利用した例
  - name: testdb                   # DB 名以外省省略した場合
```

### 4. my.cnf ファイル 設定
`my.cnf` の設定をするための変数について説明する.

| 変数　　 | デフォルト値   | コメント     | 変更の必要性 |
| :------- | :------------- | :----------- | :----------- |
| `pxc_clustering_settings` | `{}` (空のハッシュ) | クラスタ間通信に関する設定. スタンドアロンモードのときは無視される. | サーバの構成やチューニングに応じて |
| `pxc_mycnf_settings` | `{}` (空のハッシュ) | mysql の基本設定 | サーバの構成やチューニングに応じて |
| `pxc_general_settings` | `{}` (空のハッシュ) | ↑と同じく myslq の基本設定 <br> ただし, こちらは `SET GLOBAL` が可能なもの | サーバの構成やチューニングに応じて |

これらの変数は全て辞書 (ハッシュ) であり, `my.cnf` における
```ini
## my.cnf
kkk = vvv
```
といった設定を定義する際に用いる.

例えば,
```yaml
pxc_clustering_settings:
  log-bin: /var/binlog
  log-error: /var/log/mysql/error.log
```
として定義すると, `my.cnf` には
```ini
## my.cnf
log-bin = /var/binlog
log-error = /var/log/mysql/error.log
```
として設定される.

細かい定義は [defaults/main.yml](./defaults/main.yml) を参照すること.

以下に, それぞれの変数の設定例を示す.
```yaml
pxc_clustering_settings:
  wsrep_provider_options: "'gcache.dir = /var/binlog'"
  wsrep_data_home_dir: /var/binlog

pxc_mycnf_settings:
  log-bin: /var/binlog/mysql-bin
  innodb_log_group_home_dir: /var/binlog
  innodb_temp_data_file_path: /../../../var/binlog/ibtmp1:12M:autoextend
  innodb_data_file_path: /../../../var/binlog/ibdata1:12M:autoextend

pxc_general_settings:
  innodb_io_capacity: 20000
  innodb_io_capacity_max: 40000
  innodb_lru_scan_depth: 4096
  innodb_parallel_doublewrite_path: /var/binlog/xb_doublewrite
  long_query_time: 1
```

このとき, `my.cnf` には以下の設定が書き込まれる.

```ini
##clustering_settings
wsrep_provider_options = 'gcache.dir = /var/binlog'
wsrep_data_home_dir = /var/binlog

##mycnf_settings
log-bin = /var/binlog/mysql-bin
innodb_log_group_home_dir = /var/binlog
innodb_temp_data_file_path = /../../../var/binlog/ibtmp1:12M:autoextend
innodb_data_file_path = /../../../var/binlog/ibdata1:12M:autoextend

##general_settings
innodb_io_capacity = 20000
innodb_io_capacity_max = 40000
innodb_lru_scan_depth = 4096
innodb_parallel_doublewrite_path = /var/binlog/xb_doublewrite
long_query_time = 1
```

<br>

## 後方互換性について

### 削除された変数の一覧

以下の変数は `group_vars` から削除して頂いて大丈夫です.

* `percona_cluster_additional_white_ip_list`
  * ファイアーウォールの設定はすべて ufw role で実行する方針に切り替えたため削除しました.

<br>

### 変更された変数の一覧

以下の変数は名前が変更されました. 使い方は同じなので以下のように group_vars 内の変数名を修正してください.

* `percona_cluster_version` → `pxc_version`
* `percona_cluster_package_name` → `pxc_package_name`
* `percona_cluster_mycnf_path` → `pxc_mycnf_path`
* `percona_cluster_root_password` → `pxc_root_password`
* `percona_cluster_pid_file` → `pxc_pid_file`
* `percona_cluster_socket` → `pxc_socket`
* `percona_cluster_name` → `pxc_name`
* `percona_cluster_sst_user` → `pxc_sst_user`
* `percona_cluster_sst_password` → `pxc_sst_password`
* `percona_cluster_hosts` → `pxc_hosts`
* `percona_cluster_host_ips` → `pxc_host_ips`
* `percona_cluster_is_clustered` → `pxc_is_clustered`
* `percona_cluster_enable_innodb_numa_interleave` → `pxc_enable_innodb_numa_interleave`
* `percona_cluster_users` → `pxc_users`
* `percona_cluster_databases` → `pxc_databases`
* `percona_cluster_clustering_settings` → `pxc_clustering_settings`
* `percona_cluster_mycnf_settings` → `pxc_mycnf_settings`
* `percona_cluster_general_settings` → `pxc_general_settings`

<br>

## Licence
MIT
