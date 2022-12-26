# 逆向分析(靜態、動態)
## A. 靜態分析( Tool: Ghidra)
1. 寫一個c，把執行檔丟到Ghidra觀察(目前我們不知道這個程式在做什麼)
- exe檔: 要user輸入一個key
>如果隨便輸一個字串 像123的話
會提示這個key的長度在6到10之間
那再隨便輸個123456789 他會顯示錯誤
簡單來說 就是透過逆向工具知道這個程式要什麼key
``` 
Please input the key:
``` 

``` 
Please input the key:123
Error, The length of the key is 6~10.
``` 

``` 
Please input the key:123456789
Error, please input the right key.
``` 

>偷偷先解答，這個程式是一個簡單的xor運算
裡面有兩個字串 分別代表明文跟密文
- ciphertext = key ^ plaintext (XOR)
``` C++
char plaintext[10]="eastmount";
char ciphertext[10]="215244213";
``` 
- User 輸入 key, key 和 plaintext 做 XOR 存到 res, res 再和 ciphertext compare

>他是使用user輸入的key對明文字串做xor
再拿這個xor計算後的值跟密文字串做比較
一樣的話表示key輸入正確
``` C++
for(i=0;i<len;i++){
    res[i]=(key[i]^plaintext[i]);
}
``` 
2. 執行檔丟到 Ghidra 分析

![](https://i.imgur.com/LG2ZIrF.png)

- 找到main function、decompile
(不知道是不是因為用visual studio寫的關係，decompile的結果很醜、很多register)
(本來是想用IDA，但是IDA的decompile功能要IDA pro才有)
- 透過觀察可以看到剛剛說的明文密文字串
對於完全不知道這個程式在幹嘛的人而言
就是decompile 跟組語對著去看去猜

- 這張截圖當下跑的是 ciphertext = “123456789”的檔案

![](https://i.imgur.com/PivpA0w.png)

>Before: 因為只有兩個字串，又清一色都是 XOR 答案很好解
After: 多個字串(添加運算)，中間運算 decompile 之後變一大堆

![](https://i.imgur.com/YjRXjOd.png)

>在看的時候decompile那邊就是xor
而且字串還只有兩個
Xor就是三個裡面有兩個 第三個就解了嘛
所以後來我又稍微擾亂了一下
把其中一個數字的那個字串用運算湊出來

3. 再把這個版本丟進Ghidra
中間計算就會變比較複雜
就算他很快知道是xor 但對字串那邊的釐清他還是要花比較多時間

- 下面有附一個比較這兩個版本的方塊圖
後來的版本計算當然就比較多
左邊為兩個字串版本；右邊用 (字串 1 ) – (字串 2) – (字串 3)

![](https://i.imgur.com/cgJzn6t.png)

- 透過分析推測為 XOR 運算後:
就可以再寫一個程式把結果算出來
``` C++
char plaintext[10]="eastmount";
char ciphertext[10]="215244213";

len = strlen(plaintext);
for(i=0;i<len;i++){
    res[i]=(key[i]^plaintext[i]);
}
key[i]=0;
ptintf("The right key is: %s\n", key);
``` 

```
The right key is: WPFFY[G_G
``` 

```
Please input the key:WPFFY[G_G
Success, you got the right key!
``` 

## 2.動態分析 (Tool: OllyDbg)
- 踩地雷遊戲
- Search >> BeginPaint
- 設中斷點, 開始分析

![](https://i.imgur.com/IIfrZ1B.png)

>會選踩地雷是因為我比較知道我的目標要找什麼
(明顯是一個二維陣列)
那通常這種window小遊戲都會用beginpaint endpaint函數畫畫面
所以就直接先search beginpaint
然後對他設中斷點

- 在 BeginPaint 跟 EndPaint 之間有個 call function

![](https://i.imgur.com/xa9uPkI.png)

- 進去看看

![](https://i.imgur.com/d8aE5ad.png)

- 裡面一樣有很多 call，可以一個一個函數慢慢看，也可以往別路想
- 踩地雷的畫面沒有閃爍 >> 圖像應該是先有個暫存，等畫完之後再一起顯示 >> double buffer >> search BitBit, 設中斷點

![](https://i.imgur.com/RCwfJJp.png)

- 發現 BitBit 疑似在 2 層迴圈裡(二維陣列)
透過觀察得知 ESI 應該是迴圈中循環的變數、AL 影響後面顯示內容
EBX 可能是遊戲的 base register 透過+ESI 改 AL 值

![](https://i.imgur.com/ghzR6dM.png)

>發現bitbit在一個兩層的迴圈這邊，那可能就找對了
透過對這個兩層迴圈的觀察，可以得知這邊的ESI應該是迴圈裡的變量
而且裡面AL的值會影響到後面x y的值，這個al的值又是從EBX+ESI得到的
上面可以看到EBX存的是一個base address

- EBX 可能有主要資料，在 register 欄位右鍵 follow in dump 針對它看

![](https://i.imgur.com/lIeX4QU.png)

- 下方 address 顯示數據
0F 比較多，可能是安全空格
8F 較少，可能是地雷區域
若 10 為牆壁，那第一列為:0F 8F 0F 8F 0F 8F 0F 0F 0F…

![](https://i.imgur.com/DlfPz6k.png)

>主要看到有很多0F8F很像二位陣列的一個格式，中間穿插著10在裡面，
所以猜10應該是他的邊界，然後0和8，0比較多，推測0應該是好的空白格、8是地雷。
因為猜測10是邊界，所以從地雷遊戲的最左上開始橫的是 08080800


- 測試看看地雷是不是: 0F(安全) 8F(炸) 0F(安全) 8F(炸) 0F(安全) 8F(炸) …. >>推測正確
- 而且得到新的資訊，按了之後的空格:空白是40，1是41，2是42...炸彈CC

![](https://i.imgur.com/oQ9AxXk.png)

- 得到這些資訊後就可以寫程式判斷地雷位置 ex: mouse location
或再找存長寬資訊的 address, 就可以一鍵掃雷(CE)
![](https://i.imgur.com/FrAp0WM.png)


>有了這些資訊，就可以寫程式判斷地雷的位置。
像是滑鼠hover過去標題顯示是否為炸彈之類的，也可以再找長寬的資訊
我長寬的資訊是用CE去找的，就是掃描掃描掃描看變量
有長寬就可以用c之類的寫個一鍵掃雷這樣

