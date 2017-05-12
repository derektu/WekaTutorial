# XS與機器學習的整合範例

我們使用[Weka](http://www.cs.waikato.ac.nz/ml/weka/)這個系統以及XS來說明如何運用機器學習的方式來計算多種腳本條件合併的勝率。

## XS腳本

首先請大家看一下底下的警示腳本:

```
input: startDate(20110101, "開始日期");
input: days(1, "預期上漲日期");

var: forcastValue(0);

setfirstbardate(startDate);

if currentbar <= days + 5 then return;
if date = currentdate then return;  // 忽略最新一筆(盤中執行時無法知道最後漲跌幅)

once(true)
begin
    print(file("[ScriptName].arff"),
        "@RELATION demo"
    );
    print(file("[ScriptName].arff"),
        "@ATTRIBUTE f1 NUMERIC"
    );
    print(file("[ScriptName].arff"),
        "@ATTRIBUTE f2 NUMERIC"
    );
    print(file("[ScriptName].arff"),
        "@ATTRIBUTE f3 NUMERIC"
    );
    print(file("[ScriptName].arff"),
        "@ATTRIBUTE f4 NUMERIC"
    );
    print(file("[ScriptName].arff"),
        "@ATTRIBUTE f5 NUMERIC"
    );
    print(file("[ScriptName].arff"),
        "@ATTRIBUTE f6 NUMERIC"
    );
    print(file("[ScriptName].arff"),
        "@ATTRIBUTE class {-1,0,1}"
    );
    print(file("[ScriptName].arff"),
        "@DATA"
    );
end;

value1 = callfunction("rule_外資買賣超金額")[days];
value2 = callfunction("rule_外資買賣超金額")[days+1];
value3 = callfunction("rule_外資買賣超金額")[days+2];
value4 = callfunction("rule_漲跌幅")[days];
value5 = callfunction("rule_漲跌幅")[days+1];
value6 = callfunction("rule_漲跌幅")[days+2];

// forcastValue目前先設定成-1,0,1
//
value999 = rateofchange(close, days);
if value999 > 0 then forcastValue = 1;
if value999 < 0 then forcastValue = -1;
if value999 = 0 then forcastValue = 0;

print(
  file("[ScriptName].arff"),
  text(
    numtostr(value1, 0), ",",
    numtostr(value2, 0), ",",
    numtostr(value3, 0), ",",
    numtostr(value4, 0), ",",
    numtostr(value5, 0), ",",
    numtostr(value6, 0), ",",
    numtostr(forcastValue, 0)
  )
);
```

在這一個腳本內, 我們使用了當日的外資買賣超金額, 前一日的外資買賣超金額, 前兩日的外資買賣超金額, 當日漲跌幅, 前一日漲跌幅, 前兩日漲跌幅這六個欄位, 來預測明日是否會上漲.

舉例而言, 如果這一根bar是201/5/12日, 當日大盤的漲跌幅的計算方式是rateofchange(close, 1), 這時候我要抓的"當日外資買賣超金額"應該是5/11日的, "前一日的外資買賣超金額"則是5/10日的. 往前抓資料的邏輯是透過呼叫函數時多傳入offset天期, 例如 callfunction(...)(days), 表示我要取的是這個函數days之前的數值.

另外因為函數的名稱是中文字, 所以可以用callfunction()這個函數來呼叫中文函數.

底下是rule_外資買賣超金額的函數:

```
value1 = GetField("外資買賣超金額");
retval = IntPortion(value1 / 1000000000)*10;    // 每10億一等級
```

在這個函數內, 我把買賣超金額轉成10億的倍數. 如果是直接拿實際數值的話, 最後跑出來的模型應該效果會很不好.

底下是rule_漲跌幅的函數:

```
value1 = rateofchange(close, 1);
retval = IntPortion(value1 / 0.5) * 5;
```

同樣的, 我把漲跌幅每0.5%換成一個等級, 以減少運算量.

這樣子跑出來之後, 會在XS/Print目錄底下得到一個[arff](http://www.cs.waikato.ac.nz/ml/weka/arff.html)檔案, 這個檔案是weka的標準input format, 裡面除了實際數值序列之外, 還包含了欄位的定義. 範例如下:

```
@RELATION demo
@ATTRIBUTE f1 NUMERIC
@ATTRIBUTE f2 NUMERIC
@ATTRIBUTE f3 NUMERIC
@ATTRIBUTE f4 NUMERIC
@ATTRIBUTE f5 NUMERIC
@ATTRIBUTE f6 NUMERIC
@ATTRIBUTE class {-1,0,1}
@DATA
20,10,-10,0,-10,0,1
120,20,10,10,0,-10,1
70,120,20,0,10,0,1
90,70,120,0,0,10,-1
40,90,70,0,0,0,-1
20,40,90,-5,0,0,1
...
```

有了這個檔案之後, 接下來我們就可以使用weka來分析這個資料了.

## Weka操作簡介

關於Weka的教學, 網路上有非常多的支援, 大家可以找一下. 底下簡單的把流程走一遍.

- 首先打開weka程式, 點選Explorer按鈕,
![1](https://cloud.githubusercontent.com/assets/522142/25990736/6a6ec90a-3733-11e7-9ede-5a60195c9db1.png)

- 然後在Preprocess這個tab內點選Open file...按鈕, 選擇剛剛XS產生的arff檔案.
![2](https://cloud.githubusercontent.com/assets/522142/25990936/1ee8da56-3734-11e7-8a90-98e7c2a5fd4e.png),

- 接下來切換到Classify這個tab, 然後從Choose按鈕內選擇不同的分類algorithm. Weka支援非常多種不同的分類方式, 在底下這個範例內, 我們選擇SVM.
![3](https://cloud.githubusercontent.com/assets/522142/25991376/7ec917aa-3735-11e7-96b9-db76c6e26803.png).

- 選完之後, 按Start按鈕, 就可以看到分析出來的結果了.
![4](https://cloud.githubusercontent.com/assets/522142/25991377/8046de82-3735-11e7-8ad0-55dc094f62bb.png)

請注意, 這裡的勝率是直接依照我們給定的數據, 然後系統隨機分成10份, 每9份跑一次training, 用另外一份來驗證所算出來的結果(Cross validation, 10-fold的定義).
![4-1](https://cloud.githubusercontent.com/assets/522142/25991837/51510204-3737-11e7-903b-e6b9460c88ed.png)

不過, 看來這個勝率有點不行XD. 所以我們再試看看其他模型, 等待奇蹟的出現. 例如貝氏網路
![5](https://cloud.githubusercontent.com/assets/522142/25991582/43d8a1a0-3736-11e7-9f94-cf9ee945473c.png)
![6](https://cloud.githubusercontent.com/assets/522142/25991649/91de4698-3736-11e7-9f76-97ac3dbf77cb.png)

Well還是不怎麼樣, 這時候就該回去修改XS程式了XD.

## Conclusion (for now)

以上簡單的跟大家介紹利用weka這個tool來搭配XS計算出多種條件合併的勝率. 下次再跟大家介紹如何儲存算好的模型, 以及如何運用模型來預測未來的資料.

Based on 上面的流程, 我想像中, 可以把李大先前提到的idea, 變成是以下的流程:

Input:

- 使用者給定不同的條件式(每個腳本一個條件, 每個條件每期跑出一個數值)
- 使用者給定數值的分級方式(簡化成true/false, 或是每多少算一級, 或是就是直接引用計算出來的raw result)
- 使用者給定預期結果的定義, 例如是１日漲跌幅, 或是N日漲跌幅, 漲跌幅的預期可以是1/0/-1(只管上漲/下跌), 或是漲跌幅的級數
- 使用者給定回測區間

Output:

- 系統跑出每一個條件式的單獨勝率
- 系統跑出各種不同分類algorithm的組合勝率
- 使用者可以把模型儲存下來, 當有新的資料產生時, 自動回饋到系統內重新調整模型, 同時算出預期的漲跌幅


