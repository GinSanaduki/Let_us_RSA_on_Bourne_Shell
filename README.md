# Let_us_RSA_on_Bourne_Shell
RSAの仕組みをUNIXで体感する（ALL UNIX COMMAND）

* まとめたら、もう少しまともなshellを作るから、とりあえずマークダウンで書いとくぞ。
* ベースは
https://qiita.com/jabba/items/e5d6f826d9a8f2cefd60  
公開鍵暗号とRSA暗号の仕組み - Qiita  
と
https://blog.desumachi.tk/2017/10/17/%E3%82%B7%E3%82%A7%E3%83%AB%E8%8A%B8%E3%81%A7%E6%9C%80%E5%B0%8F%E5%85%AC%E5%80%8D%E6%95%B0%E3%83%BB%E6%9C%80%E5%A4%A7%E5%85%AC%E7%B4%84%E6%95%B0%E3%82%92%E6%B1%82%E3%82%81%E3%82%8B/  
シェル芸で最小公倍数・最大公約数を求める – ですまち日記帳  
を参考にした。
元ネタはRubyで書いてあるが、これは基本awk（桁あふれを起こすので、MPFR対応のgawkを使用する）で書いている。

以下のファイルを暗号化したいとする。
```bash
$ cat test.txt
Jabba the Hutto$
```

16進数にしたいので、xxdでダンプする。  
odはめんどうくさいので、後で考える・・・。  

```bash
cat test.txt | \
iconv -f "$(locale charmap)" -t utf32be | \
xxd -p | \
tr -d '\n' | \
awk -f test1.awk | \
sed -r 's/^0+/0x/' | \
xargs printf 'U+%04X\n' > XDD.txt
```

```awk
#!/usr/bin/gawk -f
# test1.awk
# awk -f test1.awk

BEGIN{
	FS="";
}

{
	for(i = 1; i <= NF; i++){
		Remainder = i % 8;
		if(i == NF || Remainder == 0){
			printf("%s\n", $i);
		} else {
			printf("%s", $i);
		}
	}
}


```

```bash
$ cat XDD.txt
U+004A
U+0061
U+0062
U+0062
U+0061
U+0020
U+0074
U+0068
U+0065
U+0020
U+0048
U+0075
U+0074
U+0074
$
```

例をカンタンにするためにそのとても大きな２つの素数をp= 7、q= 19とする。
素数は・・・なんかでまた求めるのを書いておくよ・・・。

```bash

$ p=7
$ q=19
$
$ p_minus1=$((p - 1))
$ q_minus1=$((q - 1))
$ echo $p_minus1
6
$ echo $q_minus1
18
$
```

* yesコマンドで2つの数字をスペース区切りで無限に生成する
* awkコマンドでそれぞれの数に今の行数を掛けて、間に改行を挟んで出力する
* awkコマンドで同じ数が2回目に出てきたらそれを出力して終了する

```bash
$ r=`yes $p_minus1 $q_minus1 | awk '{print $1*NR RS $2*NR}' | awk 'a[$1]++{print;exit;}'`
$ echo $r
18
```

* 公開鍵の条件は1より大きくL(=18)よりも小さい。
* しかも公開鍵とLとの最大公約数は１であること、つまり互いに素であることが条件。
* ここでは公開鍵をpublicとする。

```bash
# 並列にしてはあるが、時間がそれなりにかかるので注意
$ public=`seq 2 $r | awk -f test3.awk -v Max=$r | xargs -P 0 -r -I{} sh -c '{}' | sort -k 2n,2 -k 1n,1 | head -n 1 | cut -f 1 -d ' '`
$ echo $public
5
$
```

```awk
#!/usr/bin/gawk -f
# test3.awk
# awk -f test3.awk -v Max=$r

BEGIN{
	Max = Max + 0;
	if(Max < 2){
		exit 99;
	}
}

{
	$0 = $0 + 0;
	print "yes "$0" "Max" | awk -f test4.awk | grep -Fv --line-buffered . | awk -f test5.awk | awk -f test6.awk -v Disp="$0;
}

```

