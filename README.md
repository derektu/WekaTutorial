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

## Weka操作

首先打開weka程式, 點選[Explorer按鈕](https://cloud.githubusercontent.com/assets/522142/25990736/6a6ec90a-3733-11e7-9ede-5a60195c9db1.png), 然後在Preprocess這個tab內點選[Open file...](https://cloud.githubusercontent.com/assets/522142/25990936/1ee8da56-3734-11e7-8a90-98e7c2a5fd4e.png), 選擇剛剛XS產生的arff檔案.







