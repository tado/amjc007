AMCJ #007「実践！ライブコーディング」

# 1. 環境設定など

## 完全ライブコーディング vs. プリ・コーディング

- Alex McLeanさんは完全ライブコーディング
    - [https://youtu.be/dIpzU71LAQQ](https://youtu.be/dIpzU71LAQQ)
- Kindohmさんはあらかじめ書いたコードに書き足し?
    - [https://youtu.be/smQOiFt8e4Q](https://youtu.be/smQOiFt8e4Q)

どちらも一長一短。

- 完全ライブコーディング
    - 完全な即興から生まれるスリリングな状況の共有
    - かなりの熟練が必要
- プリ・コーディング
    - 展開の素早さ、DJ的
    - あまり作り込み過ぎると面白みがない

今回のライブはかなりプリ・コーディング寄り

## TidalCyclesライブのための環境設定

SuperColliderの起動ファイル

startup.scd

```c++
//TidalCyclesからのOSC送出(oFとの連携用)
var addr = NetAddr.new("127.0.0.1", 3333);
OSCFunc({ |msg, time, tidalAddr|
  var latency = time - Main.elapsedTime;
  msg = msg ++ ["time", time, "latency", latency];
  addr.sendBundle(latency, msg)
}, '/play2').fix;

s = Server.local;

s.reboot {
  s.options.numBuffers = 1024 * 16; //バッファサイズ
  s.options.memSize = 8192 * 16; //メモリサイズ
  s.options.sampleRate = 48000; //サンプリングレイト
  s.volume = -6.0; //音量
  s.latency = 2.0; //レイテンシー
  
  //SuperDirtの初期設定
  s.waitForBoot {
    //チャンネル数の設定
    ~dirt = SuperDirt(2, s);
    //SuperDirt付属のサンプル読み込み
    ~dirt.loadSoundFiles;
    //独自サンプル読み込み
    ~dirt.loadSoundFiles("C:/Users/tado/AppData/Local/SuperCollider/downloaded-quarks/samples-extra/*");
    //bootするまで待機
    s.sync;
    //SuperDirt起動
    ~dirt.start;
  }
}
```

ポイント

- OSCによるoFとの連携
- SuperColliderが起動したらすぐにSuperDirtを起動するように
- SuperDirtのデフォルトのサンプルを読み込んだ後で、自分で追加したオリジナルサンプル読み込み
    - こちら! → [https://github.com/tado/samples-extra](https://github.com/tado/samples-extra)

Windows 10のTips

- 起動して最初のbootに時間がかかる場合
- Windows Securityの例外にSC関連のプロセスを入れておく

![](./img/screen1.jpg)

# 2. 基本編: 単純なパターンから進化させる

ライブの際のコードを説明する前に、基本事項の確認

ノリ(グルーブ)の源泉、シンコペーション

```haskell
d1 $s "bd hc bd cp" #cps(70/120) -- シンプルすぎる!

d1 $s "bd hc [~ bd] cp" -- シンコペーションによるグルーブ

d1 $s "bd [~ hc] [~ bd] cp" -- さらにシンコペーション

d1 $s "bd [~ hc] [~ bd] [~ [~ cp]]" -- やりすぎは良くない
```

ループ単位で変化させる

```haskell
d1 $s "bd <~ hc [~ hc]> [~ bd] <cp cp*4 ~ [hc*2 ho]>" -- <>の中を順番に
```

2つのパートの共存

```haskell
d1 $s "{bd hc [~ bd] cp, ho [~ ho] ho [~ ho*2]}"
```

パターンの上部でリズムに変化をつける

- jux, brak, fast, slowなど

```haskell
d1
  $jux (iter 8)
  $s "{bd hc [~ bd] cp, ho [~ ho] ho [~ ho*2]}"
```

変化のタイミングを設定

- sometimes, sometimesBy, every など

```haskell
d1
  $sometimesBy 0.3 (jux (iter 8))
  $sometimes brak
  $every 4 (fast 2)
  $s "{bd hc [~ bd] cp, ho [~ ho] ho [~ ho*2]}"
```

パターンの下でエフェクトをつける

- delay, room, shape, gainなど

```haskell
d1
  $every 6 (slow 2)
  $sometimesBy 0.3 (jux (iter 8))
  $every 4 (fast 2)
  $s "{bd hc [~ bd] cp, ho [~ ho] ho [~ ho*2]}"
  #delay "0.5" #delayt "1.5" #delayfb "0.5"
  #shape "0.5"
  #cps(70/120)
```

フィルター設定

```haskell
d1
  $sometimesBy 0.3 (jux (iter 8))
  $every 4 (fast 2)
  $s "{bd hc [~ bd] cp, ho [~ ho] ho [~ ho*2]}"
  #delay "0.5" #delayt "1.5" #delayfb "0.5"
  #lpf (range 100 10000 $slow 4 $sine)
  #resonance "0.2"
  #shape "0.7"
```

変拍子、ポリリズム、ポリミーター

変拍子 (4で割り切れないリズム)

```haskell
d1 $s "bd hc bd cp cp" -- 5拍子

d1 $s "bd ~ hc bd cp ~ cp" -- 7拍子
```

変拍子にリズム変化

```haskell
d1
  $sometimesBy 0.3 (rev)
  $jux (iter 14)
  $s "bd ~ hc bd cp ~ cp" -- 7拍子
```  

ポリリズム、複数のリズムの共存

```haskell
d1 $s "[bd ~ hc bd cp ~ cp, bd hc bd ho]" -- 7 + 4拍子
```

ポリリズム + リズム変化

```haskell
d1 
  $every 3 (rev)
  $sometimesBy 0.3 (jux (iter 14))
  $s "[bd ~ hc bd cp ~ cp, bd hc bd ho]"
  #lpf (range 100 10000 $slow 4 $sine)
  #resonance "0.2"
```  

ポリミーター、複数のリズムのタイミングがずれていく

```haskell
d1 $s "{bd ~ hc*2 bd cp ~ cp*3, bd ~ hc cp}" -- 7 + 4拍子
```

ポリミーター + リズム変化

```haskell
d1 
  $every 3 (rev)
  $sometimesBy 0.3 (jux (iter 14))
  $sometimesBy 0.4 (slow 2)
  $s "{bd ~ hc*2 bd cp ~ cp*2, bd hc hc cp}"
  #lpf (range 100 10000 $slow 4 $sine) #resonance "0.2"
  #shape "0.5"
  #cps(75/120)
```

ユークリッドリズム(便利!!)

```haskell
d1 $s "cp(3, 8)"

d1 $s "cp(5, 8)"
``` 

ユークリッドリズムの理屈

- (3, 8)の場合
  - [1 1 1 0 0 0 0 0] (3, 8)
  - [1 0][1 0][1 0][0 0] 1と0をペアに
  - [1 0 0][1 0 0][1 0] 右に余った0を左から順に入れていく
  - [1 0 0 1 0 0 1 0] 完成


- (5, 8)の場合
  - [1 1 1 1 1 0 0 0]
  - [1 0][1 0][1 0][1 1] 1と0をペアに
  - [1 0 1][1 0 1][1 0] 余った1を配分
  - [1 0 1 1 0 1 1 0] 完成

3つめのパラメータでシフト

```haskell
d1 $s "cp(5, 8, 1)"

d1 $s "cp(5, 8, 2)"

d1 $s "cp(5, 8, 3)"
```

2つのユークリッドリズムを重ねることで複雑なリズムに

```haskell
d1 $s "{cp(5, 8, 3), bd(3, 8, 0)}"

d1 $s "{cp(9, 14, 5), bd(4, 14, 0)}" -- 7(14)拍子
```

stackで音を積む

```haskell
d1
  $stack
  [
    s "cp(9, 14, 5)",
    s "hc(11, 14, 3)",
    s "ho(3, 14, 10)",
    s "bd(4, 14, 0)"
  ]
  #cps(70/120)
```

stackの上でパターン変化、下にエフェクト

```haskell
d1
  $every 4 (slow 2)
  $sometimesBy 0.7 (jux (iter 14))
  $sometimesBy 0.2 (rev)
  $stack
  [
    s "cp(9, 14, 5)",
    s "hc(11, 14, 3)",
    s "ho(3, 14, 10)",
    s "bd(4, 14, 0)" #gain "1.2"
  ]
  #lpf (range 80 20000 $rand)
  #resonance "0.3"
  #shape "0.5"
  #cps (70/120)
```

シンセの音でやっても面白い

```haskell
d1
  $sometimesBy 0.7 (jux (iter 14))
  $sometimesBy 0.2 (rev)
  $stack
  [
    s "supersaw(11, 14, 5)" #note 12,
    s "supersaw(11, 14, 3)" #note 7,
    s "supersaw(9, 14, 0)" #note 0
  ]
  |+| note "[0, -5, -12]"
  |-| note "[12, 0]"
  #sustain "0.08"
  #lpf (range 80 20000 $slow 16 $sine)
  #resonance "0.4"
  #shape "0.5"
  #delay "0.5" #delayt "0.25" #delayfb "0.75"
  #cps (70/120)
```

# 3. 応用編: 実際のコードをみながら

### 0. 全体の設定

```haskell
-- 7拍子、 75/120cps
d1
  $ s "uni:1(1, 7)"
  # gain "0.6"
  # cps(75/120)

-- イントロ
-- 長いサンプルをサンプリング
-- ピッチをずらして厚みをつける

d1
  $slow 4
  $s "matsu"
  #gain "0.7"
  #n "[2.0, -1.5, 1.0]"
```

### 1. 冒頭のパート

```haskell
-- コード(和音)
do
  let chord = "d'sus2"
  d2
	  -- $jux ((3/7)~>)
    $stack
    [
			-- s "supersaw(5,14,5)" #note chord |+|note "7" #gain "0.8",
      s "supersaw(5,14,0)" #note chord |+|note "0" #gain "1.0"
    ]
    #sustain "0.1"
    |*| gain "0.7"
    -- |-| note "[0, 12]"
    -- #cutoff (range 800 18000 $slow 16 $sine) #resonance "0.3"
    -- #pan (choose [-0.5, 0.0, 0.5])
    -- #room "0.3" #size "0.9"
    -- #shape "0.5"

-- ベース(旋律)
d3
  $ s "superbass(14, 14)"
  # sustain "0.1"
  # note ((scale "minPent" "{0 .. 7}%9"))
  |-| note "10"
  # gain "0.8"
  # lpf (range 100 4000 $rand)
  # resonance "0.3"

-- minPent majPent ritusen egyptian kumai hirajoshi iwato chinese indian pelog prometheus scriabin gong shang jiao zhi yu whole augmented augmented2 hexMajor7 hexDorian hexPhrygian hexSus hexMajor6 hexAeolian major ionian dorian phrygian lydian mixolydian aeolian minor locrian harmonicMinor harmonicMajor melodicMinor melodicMinorDesc melodicMajor bartok hindu todi purvi marva bhairav ahirbhairav superLocrian romanianMinor hungarianMinor neapolitanMinor enigmatic spanish leadingWhole lydianMinor neapolitanMajor locrianMajor diminished diminished2 chromatic
```

### 2. 最初のビート

```haskell
d4
  $jux ((3/14) <~)
  $s "uni(5, 14)"
  #gain "1.1"

d5
  $jux (iter 7)
  $s "uni(11, 14)" 
  #n "{2 3 1}%5"
  #gain "1.1" #shape "0.5"

  #delay "0.6"
  #delayt (choose([0.005, 0.02, 0.01, 0.025])) 
  #delayfb "0.5"
  #orbit 1
```


### 3. 最初の旋律

```haskell
d1
  $sound "tet(9, 14, [0, 2])"
  #gain "1.0"

  #sustain "0.1"
  #n "{0 .. 3}%9"
  #lpf (range 800 10000 $rand)  #resonance "0.3"
  #up "{[0,7] [7,14]}%9"

d3
  $jux ((3/7) <~)  
  $sound "jimsyn(3, 7)"
  #n "<20 1>"
  |*| up "-0.2"
  |*| gain "1.0"
  #lpf (range 2000 8000 $slow 12 $sine)
  #resonance "0.3"

do 
  let chord1 = "c'sus2" 
  d2
    $stack
    [
      -- sound "supersaw(4,14,10)" # note chord1 |+|note "12", 
      -- sound "supersaw(4,14,5)" # note chord1 |+|note "7",
      sound "supersaw(4,14,0)" # note chord1 |+|note "0"
    ]
    #sustain "0.08"
    #gain "0.9"
    #lpf (range 800 8000 $slow 16 $sine)
    #resonance "0.2"

hush
```

### 4. 2番目のビート

```haskell
d1
  $slow 4
  $s "matsu"
  #n "[1.0, -1.5]"
  #gain "1.3"

d1
  $jux ((3/7) ~>)
  $s "deepsyn(5, 14)"
  #gain "1.0"
  #n "<6 7 8 9>*4"
  #up "[19, 24]"
  #speed "[1.01, 1.0]"
  #lpf (range 300 8000 $slow 14 $sine)
  #resonance "0.3"

d4
  -- $sometimesBy 0.5 ((3/14) <~)
  $stack
  [
    -- s "tabla2(5, 14, {0})" #gain "1.3" #n (irand 30),
    -- s "kon(3, 7, 0)" #n 0 #gain "1.2",
    s "uni(9, 14, 2)" #n 1 #gain "1.2",
    s "uni(5, 14)" #gain "1.6"
  ]
  |*|gain "1.0"
  -- #hpf 4000
```


### 5. コードの盛り上り!!

```haskell
d5
  $jux (rev)
  $s "supersaw(14, 14)"
  #sustain "0.1"
  # n (scale "minPent" "{0 .. 7}%11") |-| n "12"
  |*| n "[1.0, 1.008]"
  -- |+| n "[0, 12]"
  #voice (range 0.1 0.2 $slow 24 $sine)
  #cutoff (range 100 18000 $slow 14 $sine)
  #resonance "0.2"
  #gain "1.0"
  -- #semitone "{7 0 12 5 19 24}%13"
```

### 6. ソロにしてから激しいビートへ

```haskell
solo 5

d2
  $stack
  [
    s "gabba" #gain "1.2",
    s "glitch(6,14,3)" #n (irand 64),
    s "gabba(6,14,0)" #n (irand 64),
    s "ifdrums(10,14)" #n (irand 64)
  ]
  -- #hpf 1000
  |*|gain "1.3"
  #shape "0.5"

d3
  $jux ((5/14) ~>)
  $s "uni(5, 14, [0, 6, 9, 12])"
  #n "{0 1 2 3 4}%5"
  -- #delay "0.5"
  -- #delaytime "{0.005 0.02 0.01}%4"
  -- #delayfeedback "0.9"
  #gain "1.3"
  #shape "0.5"

d2
  $every 8 (jux (rev))
  $sometimesBy 0.2 ((3/7) <~)
  $stack
  [
    s "distd(2, 7)", 
    s "{bd cp bd hc}%7" #n (irand 12),
    s "uni(5, 14, {0, 3})"
    #n "0 0 0 1 0 3 1 2"
  ]
  #gain "<1 1 1 0>"
  |*|gain "1.3"
  #shape "0.5"  

d2
  -- $sometimes (jux (iter 14))
  -- $sometimes (jux (iter 7))
  $sometimes (rev)
  $sound "{ifdrums(9, 14, 0), bd(3, 14, 0), gabba(3, 14, 4)}"
  #gain "1.3"
  #shape "0.9"
  #n (irand 64)
  #pan (rand)
```  


### 7. 最後の盛り上がり!!

```haskell
do
  d5 silence
  d6 silence
  let
    pat1 = "{0 ~ ~ 0 ~ ~ 0 ~ 0 0 ~ 0 ~ ~}%14"
  d1
    $sometimesBy 0.1 (rev)
    $sometimesBy 0.4 ((1/7) <~)
    $stack
    [
      s "uni(11, 14)" #n "{0 1 2}%7", 
      s "distd(2, 14)" |*|gain "1.8" #voice "4" #sustain "0.2",
      s "uni:1(2, 14, 5)"
    ]
    #shape "0.9"
    |*|gain "1.5"
  d2
    $every 3 (jux ((3/7) ~>))
    $stack
    [
      up pat1
      # s "bfm"
      # note (choose [12, 0, -12, -24, -36])
      |+| note "{0, 5, 7, 9}"
    ]
    #sustain (choose [0.05, 0.12, 0.15])
    #pitch1 (choose [0.33, 3.33, 19.33])
    #voice (choose [30, 1000, 4000, 12000])
    #delay "0.8"
    #delaytime (choose [0.01, 0.03, 0.02, 0.008])
    #delayfeedback "0.8"
    |*|gain "1.5"
    #room "0.4"
    #size "0.2"

do
  let
    pat2 = "{0 ~ 0 ~ ~ 0 ~ 0 ~ ~}%14"
  d3
    $every 4 (jux ((1/7) ~>))
    $up pat2
    # s "superzow"
    # note (choose [19, 12, 0])
    |+| note "{0, 7, 11}"
    #sustain (choose [0.08, 0.12, 0.3])
    #shape "0.8"
    #lpf (range 1000 20000 $rand)
    #resonance "0.4"
    #detune "10"
    #accelerate (choose [0, 0, 1, 4]) #slide "1"
    #gain "1.5"

do
  let
    pat3 = "{0 ~ 0 ~ ~ 0 0 ~ 0}%14"
  d4
    $sometimesBy 0.4 (jux ((1/7) ~>))
    $up pat3
    # s "supersiren"
    # sustain "0.15"
    # note "[0,5,7,9,11]"
    |+| note (choose [0, 5, 7, 9, 11])
    |+| note (choose [-12, -24, 0, 12, 24])
    #pan (rand)
    #gain "1.9"

d5
  $s "{superzow*14, uni:1*14}"
  #n "{0,5,7,9,11}"
  |+|n "{-12,0,12,24,48}"
  #sustain "0.08"
  #gain (range 0.0 2.5 $slow 24 $saw)
  -- #vowel "{a i u e o}%12"
  -- #room "0.2" #size "0.2"
```

### 9. フィナーレ!!

```haskell
do
  let chord = "d'sus4"
  d5
	  $jux ((3/7)~>)
    $stack
    [
			s "supersaw(5,14,5)" #note chord |+|note "7" #gain "0.8",
      s "supersaw(5,14,0)" #note chord |+|note "0" #gain "1.0"
    ]
    #sustain "0.1"
    |*| gain "1.5"
    #room "0.3" #size "0.9"
    |-| note "[0, 12]"
    #cutoff (range 800 18000 $slow 16 $sine)
    #resonance "0.3"
		#pan (choose [-0.5, 0.0, 0.5])
    #shape "0.7"

d3
  $ s "superbass(14, 14)"
  # sustain "0.01"
  # n ((scale "yu" "{0 .. 12}%5") - 10)
  # gain "1.0"
  # lpf (range 100 1000 $rand)
  # resonance "0.3"

d4
  $slow 2
  $jux ((3/7) ~>)
  $sound "autech2(5, 14, [0, 5, 10])"
  #gain "1.4"
  #speed "[1.005, 1.0]"
  #n "{0 1 2 3}%5"
  #lpf (range 4000 18000 $slow 4 $sine) #resonance "0.3"
  |*|speed "[-1.0, 3]"
  #shape "0.5"

d3
  $jux (rev)
  -- $sometimesBy 0.2 (jux ((3/8)~>))
  -- $sometimesBy 0.1 ((3/8) <~)
  $stack
  [
    s "uni(6, 14)" #gain "1.7" #shape "0.9",
    sometimes (jux ((5/7) ~>))
    $s "uni(5, 14, [0, 6, 9, 12])"
    #note "{1 2 3}%6"
    #gain "1.2"
    #up "{0 -7 7 12}%7"
  ]
  -- |*| gain "<1 1 1 1 1 1 1 0>"
  |*| gain "0.5"
  #shape "0.8"
  -- #hpf 1000

d1
  $jux ((5/7) ~>)
  $s "sfh(1, 14)"
  #gain "1.5"
  #n (irand 200)
  #pan (range 0.4 0.6 $rand)
  #up "[7, 0, 12]"
  #speed "[1.01, 1.0]"

d2
  $slow 8
  $sound "empty"
  #speed "[1.0, 1.01]"
  #gain "1.5"
  #lpf "12800"

d5
  $slow 4
  $s "matsu"
  #n "[1.0, -1.5]"
  #gain "1.0"  

d9 silence

unsolo 5

hush
```

おしまい！！