マルチプルテーブル
# 課題内容
スイッチ動作の各ステップについて、trema dump_flows の出力 (マルチプルテーブルの内容) を混じえながら動作を説明する．
#動作確認方法
まず，tremaを，設定confファイル，コントローラrbファイルをもとにopenflow13を利用して実行するには，次のコマンドを実行する．

```
./bin/trema run lib/learning_switch13.rb --openflow13 -c trema.conf
```

# 初期化
例外的なパケットや初期段階で決まっているパケットの取扱いなどの基本的なフローをフローテーブルに設定する．
そもそもフローテーブルは２つに別れており，イングレスフィルタリングテーブルとフォワーディングテーブルである．前者では基本的にパケットをドロップさせるか，通過させるかを判断する．

- add_multicast_mac_drop_flow_entry関数
イングレステーブルにおいて，パケットの宛先がマルチキャストを表す'01:00:00:00:00:00'であった場合は，dropさせ，届かないようにする．
- add_ipv6_multicast_mac_drop_flow_entry関数
先程の関数のIPv6版である．
- add_default_broadcast_flow_entry関数
フォワーディングテーブルにおいて，パケットの宛先がブロードキャストアドレスである'ff:ff:ff:ff:ff:ff'であった場合，フラッディングを行うようなinstructionを設定する．
- add_default_flooding_flow_entry関数
フォワーディングテーブルにおいて，残ったパケットはコントローラに送信する．優先度は最も低いので，他にフローテーブルが定義されればそれに従う．
- add_default_forwarding_flow_entry関数
イングレステーブルにおいて，ドロップされなかったパケットはフォワーディングテーブルに行く．


# パケットへの対応
host1とhost2を用意し，互いにパケットを送信する．そのときパケットが届くか，フローテーブルがどうなるかを示す．
## 初期状態のフローテーブル

```
cookie=0x0, duration=3.406s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=3.367s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=3.367s, table=0, n_packets=3, n_bytes=1026, priority=1 actions=goto_table:1
cookie=0x0, duration=3.367s, table=1, n_packets=3, n_bytes=1026, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=3.367s, table=1, n_packets=0, n_bytes=0, priority=1 actions=CONTROLLER:65535
```

このようになっており，プログラムされたもののみが記録されている．
## host1からhost2へパケットを送信したとき

```
trema show_stats host2
Packets received:
  192.168.0.1 -> 192.168.0.2 = 1 packet
```

となり，パケットは届くが，フローテーブルは通過パケット数・サイズの値しか変化しない．
理由としては，このパケットはフローテーブルをすべて通過し，コントローラにpacket_inするが，host1のアドレスとポート番号を記録するのみだからである．
続いて次の試行を行う．

## host2からhost1へパケットを送信したとき
このときもパケットは無事に届くが，フローテーブルは次のように変化する
'''
cookie=0x0, duration=8.806s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=8.768s, table=0, n_packets=20, n_bytes=3526, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=8.768s, table=0, n_packets=5, n_bytes=1110, priority=1 actions=goto_table:1
cookie=0x0, duration=8.768s, table=1, n_packets=3, n_bytes=1026, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=3.213s, table=1, n_packets=0, n_bytes=0, idle_timeout=180, priority=2,in_port=2,dl_src=e6:0d:d1:f3:ac:ca,dl_dst=bb:37:78:ff:2a:a8 actions=output:1
cookie=0x0, duration=8.768s, table=1, n_packets=2, n_bytes=84, priority=1 actions=CONTROLLER:65535
'''
このとき，５つめのルールに，ポート2でのhost1とhost2のホスト間のpacket_outを記録している．
なぜなら，host2からのパケットが届きpacket_inしたとき，このポート番号は既知であり，add_forwarding_flow_entryが実行される．その結果packet_outが実行される．

## 再びhost1からhost2へパケットを送信したとき
フローテーブルにはさきほど追加されたものと逆のルールが保存される．なぜなら，host2のポートは既知であるから，同じようにpacket_outが実行されるからである．
