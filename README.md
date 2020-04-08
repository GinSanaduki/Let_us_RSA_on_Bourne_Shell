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

