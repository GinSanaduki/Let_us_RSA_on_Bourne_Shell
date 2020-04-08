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




* 公開鍵は「$p_qで割ったの余りの表を使って$public乗すること」になる。
```bash
$ p_q=$((p * q))
$ echo $p_q
133
$
```

* 秘密鍵の条件は
* 秘密鍵✕公開鍵＝L * N + 1となる数。
* ここでは秘密鍵をprivateとする。

```bash
private=`seq 2 $r | awk -f test7.awk -v Max=$r -v Public=$public`
echo $private
```

```awk
#!/usr/bin/gawk -f
# test7.awk
# awk -f test7.awk -v Max=$r -v Public=$public

BEGIN{
	Max = Max + 0;
	Public = Public + 0;
}


{
	$0 = $0 + 0;
	Public_ColZero = Public * $0;
	Remainder = Public_ColZero % Max;
	if(Remainder == 1){
		print;
		exit;
	}
}


```

* 冒頭のJaba the Huttを公開鍵（public=5）で暗号化する。

元の数字を5回かけて、133の余りを出す。

$ /usr/bin/gawk -M -f test8.awk -v Max=$p_q -v Exponentiation=$public Conv_XDD.csv | \
tr -d '\n' | \
awk '{print substr($0,1,length($0) - 1);}' > Encrypt.csv

# stderr
i : 1, $i : 74
Exponentiation : 5
Exp : 2219006624
Max : 133
Remainder : 44
i : 2, $i : 97
Exponentiation : 5
Exp : 8587340257
Max : 133
Remainder : 13
i : 3, $i : 98
Exponentiation : 5
Exp : 9039207968
Max : 133
Remainder : 91
i : 4, $i : 98
Exponentiation : 5
Exp : 9039207968
Max : 133
Remainder : 91
i : 5, $i : 97
Exponentiation : 5
Exp : 8587340257
Max : 133
Remainder : 13
i : 6, $i : 32
Exponentiation : 5
Exp : 33554432
Max : 133
Remainder : 128
i : 7, $i : 116
Exponentiation : 5
Exp : 21003416576
Max : 133
Remainder : 51
i : 8, $i : 104
Exponentiation : 5
Exp : 12166529024
Max : 133
Remainder : 111
i : 9, $i : 101
Exponentiation : 5
Exp : 10510100501
Max : 133
Remainder : 5
i : 10, $i : 32
Exponentiation : 5
Exp : 33554432
Max : 133
Remainder : 128
i : 11, $i : 72
Exponentiation : 5
Exp : 1934917632
Max : 133
Remainder : 116
i : 12, $i : 117
Exponentiation : 5
Exp : 21924480357
Max : 133
Remainder : 129
i : 13, $i : 116
Exponentiation : 5
Exp : 21003416576
Max : 133
Remainder : 51
i : 14, $i : 116
Exponentiation : 5
Exp : 21003416576
Max : 133
Remainder : 51

$cat Encrypt.csv
44,13,91,91,13,128,51,111,5,128,116,129,51,51
$

```

```awk
#!/usr/bin/gawk -f
# test8.awk
# /usr/bin/gawk -M -f test8.awk -v Max=$p_q -v Exponentiation=$public Conv_XDD.csv > Encrypt.csv
# /usr/bin/gawk -M -f test8.awk -v Max=$p_q -v Exponentiation=$private Encrypt.csv > Decrypt.csv

BEGIN{
	FS = ",";
	Max = Max + 0;
	Exponentiation = Exponentiation + 0;
	# print Max;
	# print Exponentiation;
}

{
	delete Arrays;
	for(i = 1; i <= NF; i++){
		print "i : "i", $i : "$i > "/dev/stderr";
		Exp = $i ** Exponentiation;
		print "Exponentiation : "Exponentiation > "/dev/stderr";
		print "Exp : "Exp > "/dev/stderr";
		Remainder = Exp % Max;
		print "Max : "Max > "/dev/stderr";
		print "Remainder : "Remainder > "/dev/stderr";
		print Remainder"," > "/dev/stdout";
	}
}

```

* 逆パターンとして、秘密鍵の11と133を使って復号する。
* 暗号の数字を11回かけて、133の余りを出す。

```bash
$ /usr/bin/gawk -M -f test8.awk -v Max=$p_q -v Exponentiation=$private Encrypt.csv | \
tr -d '\n' | \
awk '{print substr($0,1,length($0) - 1);}' > Decrypt.csv
# stderr
i : 1, $i : 44
Exponentiation : 11
Exp : 1196683881290399744
Max : 133
Remainder : 74
i : 2, $i : 13
Exponentiation : 11
Exp : 1792160394037
Max : 133
Remainder : 97
i : 3, $i : 91
Exponentiation : 11
Exp : 3543686674874777831491
Max : 133
Remainder : 98
i : 4, $i : 91
Exponentiation : 11
Exp : 3543686674874777831491
Max : 133
Remainder : 98
i : 5, $i : 13
Exponentiation : 11
Exp : 1792160394037
Max : 133
Remainder : 97
i : 6, $i : 128
Exponentiation : 11
Exp : 151115727451828646838272
Max : 133
Remainder : 32
i : 7, $i : 51
Exponentiation : 11
Exp : 6071163615208263051
Max : 133
Remainder : 116
i : 8, $i : 111
Exponentiation : 11
Exp : 31517572945366073781711
Max : 133
Remainder : 104
i : 9, $i : 5
Exponentiation : 11
Exp : 48828125
Max : 133
Remainder : 101
i : 10, $i : 128
Exponentiation : 11
Exp : 151115727451828646838272
Max : 133
Remainder : 32
i : 11, $i : 116
Exponentiation : 11
Exp : 51172646912339021398016
Max : 133
Remainder : 72
i : 12, $i : 129
Exponentiation : 11
Exp : 164621598066108688876929
Max : 133
Remainder : 117
i : 13, $i : 51
Exponentiation : 11
Exp : 6071163615208263051
Max : 133
Remainder : 116
i : 14, $i : 51
Exponentiation : 11
Exp : 6071163615208263051
Max : 133
Remainder : 116
$ cat Decrypt.csv
74,97,98,98,97,32,116,104,101,32,72,117,116,116
```

* ダンプから起こしたunicode pointと突合する

```bash
$ diff -q Decrypt.csv Conv_XDD.csv
$ echo $?
0
$
```

# まあ、こんな感じだ。
# bashでも、できてよかったね。

