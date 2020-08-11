# bootstrap.yml について

## 概要
[bootstrap.yml](../tasks/bootstrap.yml) を実行する時は, 他にドナーノードが存在するか, そして `grastate.dat` の `safe_to_bootstrap` の値がどうなっているかということに注意をしなくてはならない. 

また, ブートストラップ時の処理について詳しく知るためにも, 
[Galera レプリケーション - PXC クラスタを回復する方法](https://www.percona.com/blog/2014/09/01/galera-replication-how-to-recover-a-pxc-cluster)
を読んでおくことが望ましい. 

この README ではこれらの状態とブートストラップ時のケースについてまとめている. 

## 前提
前提として以下のホスト名のマシンが DB クラスタとして存在する. 
- infra-a
- infra-b
- infra-c

## 各クラスタの状態におけるブートストラップ処理について

ブートストラップ処理が実行されるべき条件を以下に示す.

1. `grastate.dat` 内の `safe_to_bootstrap` の値が 1 である.
2. クラスタ内にドナーノードが存在していない.
3. 2台以上のノードで1の条件を満たす場合, その中のノードの内1台をブートストラップさせるノードとする.

### ケース 1: infra-a が正常に終了し, infra-{b,c} が稼働している場合
#### infra-{b,c} について
1. infra-{b,c} の `grastate.dat` 内の `safe_to_bootstrap` の値は 0 である.
2. infra-{b,c} が稼働しているため, クラスタ内にドナーノードは存在している.
3. 条件1.を満たすノードは0台である.

全ての条件を満たさなかったため, infra-{b,c} は両方共ブートストラップ処理しない.

#### infra-a について
1. infra-a の `grastate.dat` 内の `safe_to_bootstrap` の値は 0 である.
2. infra-{b,c} が稼働しているため, クラスタ内にドナーノードは存在している.
3. 条件1.を満たすノードは0台である. 

全ての条件を満たさなかったため, infra-a はブートストラップ処理しない. 


### ケース 2: infra-{a,b} が正常に終了し, infra-c が稼働している場合
#### infra-c について
1. infra-c の `grastate.dat` 内の `safe_to_bootstrap` の値は 1 である.
2. infra-c が稼働しているため, クラスタ内にドナーノードは存在している.
3. 条件1.を満たすノードは1台である (infra-c).

1と3は満たしているが, 2を満たさなかったため, infra-c はブートストラップ処理しない. 

#### infra-{a,b} について
1. infra-{a,b} の `grastate.dat` 内の `safe_to_bootstrap` の値は 0 である.
2. infra-c が稼働しているため, クラスタ内にドナーノードは存在している.
3. 条件1.を満たすノードは1台である (infra-c).

1 と 2 の条件を満たさなかったため, infra-{a,b} はブートストラップ処理しない.


### ケース 3: infra-{a,b,c} が全て正常に終了している場合
例えば, infra-b の `grastate.dat` 内の `safe_to_bootstrap` の値が 1 であるとする.
#### infra-b について
1. infra-b の `grastate.dat` 内の `safe_to_bootstrap` の値が 1 である.
2. どのノードも動いていないため, クラスタ内にドナーノードは存在していない.
3. 条件1.を満たすノードは1台である (infra-b)

全ての条件を満たしているため, infra-b はブートストラップ処理をする. 

#### infra-{a,c} について
1. infra-{a,c} の `grastate.dat` 内の `safe_to_bootstrap` の値が 0 である. 
2. どのノードも動いていないため, クラスタ内にドナーノードは存在していない.
3. 条件1.を満たすノードは1台である (infra-b)

1 の条件を満たさなかったため, infra-{a,c} はブートストラップ処理しない.


### ケース 4: infra-{a,b,c} に初めて Percona XtraDB Cluster をインストールした場合
1. infra-{a,b,c} の `grastate.dat` 内の `safe_to_bootstrap` の値は 1 である.
2. どのノードも動いていないため, クラスタ内にドナーノードは存在していない. 
3. 1. の条件を満たすノードは3台である (infra-{a,b,c})

1 と 2 の条件を満たしている. また 3 の条件により, infra-{a,b,c} 内の 1 台をブートストラップさせる. 

※ 例えば, infra-a のみブートストラップさせる

### ケース 5: infra-{b,c} が稼働していて, infra-a には初めてインストールした場合
#### infra-{b,c} について
1. infra-{b,c} の `grastate.dat` 内の `safe_to_bootstrap` の値は 0 である.
2. infra-{b,c} が稼働しているため, クラスタ内にドナーノードは存在している.
3. 条件1.を満たすノードは0台である.

全ての条件を満たさなかったため, infra-{b,c} は両方共ブートストラップ処理しない.

#### infra-a について
1. infra-a の `grastate.dat` 内の `safe_to_bootstrap` の値は 1 である.
2. infra-{b,c} が稼働しているため, クラスタ内にドナーノードは存在している.
3. 条件1.を満たすノードは1台である (infra-a)

2 の条件を満たさないため, infra-a はブートストラップ処理しない.

### ケース 6: infra-b が稼働中, infra-c が正常に終了中, infra-a には初めてインストールした場合
#### infra-b について
1. infra-b の `grastate.dat` 内の `safe_to_bootstrap` の値は 1 である.
2. infra-b が稼働しているため, クラスタ内にドナーノードは存在している.
3. 条件1.を満たすノードは2台である (infra-{a,b}).

2 の条件を満たさないため, infra-b はブートストラップ処理しない.

#### infra-c について
1. infra-c の `grastate.dat` 内の `safe_to_bootstrap` の値は 0 である.
2. infra-b が稼働しているため, クラスタ内にドナーノードは存在している.
3. 条件1.を満たすノードは2台である (infra-{a,b}).

1 と 2 の条件を満たさないため, infra-c はブートストラップ処理しない.

#### infra-a について
1. infra-a の `grastate.dat` 内の `safe_to_bootstrap` の値は 1 である.
2. infra-b が稼働しているため, クラスタ内にドナーノードは存在している.
3. 条件1.を満たすノードは2台である (infra-{a,b}).

2 の条件を満たさないため, infra-a はブートストラップ処理しない.

## その他
上記のケース5とケース6は後からノード2台構成のクラスタに後からノード1台を参加させる例である. 
