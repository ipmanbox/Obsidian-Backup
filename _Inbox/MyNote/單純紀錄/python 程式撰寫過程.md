age = 28
if (18 <= age):
    print('賣票給它')
    
賣票給它



age = 70
if (60 <= age):
    print('票價為200元')
    elif (18 <= age):
    print('票價為400元')
    else:
    print('不賣票給他')
  Cell In[12], line 5
    print('票價為400元)
          ^
SyntaxError: unterminated string literal (detected at line 5)


age = 70
if (60 <= age):
    print('票價為200元')
    elif (18 <= age):
    print('票價為400元')
    else:
    print('不賣票給他')
  Cell In[13], line 4
    elif (18 <= age):
    ^
SyntaxError: invalid syntax


age = 70
if (60 <= age):
    print('票價為200元')
elif (18 <= age):
    print('票價為400元')
else:
    print('不賣票給他')
    
票價為200元


age = 50
if (60 <= age):
    print('票價為200元')
elif (18 <= age):
    print('票價為400元')
else:
    print('不賣票給他')
    
票價為400元


age = 15
if (60 <= age):
    print('票價為200元')
elif (18 <= age):
    print('票價為400元')
else:
    print('不賣票給他')
    
不賣票給他





注意if[語法正確，但邏輯順序錯誤]的問題

if的程式判斷方式是由上而下，依序執行。
若是一開始就判斷年齡超過18歲(滿足判斷條件)，判斷為true。就會直接輸出'票價為400元'。
後面的程式，就不會繼續判斷了。

順序錯誤的程式:

age = 70
if (18 <= age): <-會先判斷是否18歲(包含)以上
    print('票價為400元')
elif (60 <= age):<-再判斷是否60歲(包含)以上
    print('票價為200元')
else:
    print('不賣票')

票價為400元


可以將語法順序修改成以下這樣:

age = 70
if (18 <= age <= 59):
    print('票價為400元')
elif (60 <= age):
    print('票價為200元')
else:
    print('不賣票')
    
票價為200元




請chatgpt，修改語法:
[我]
以下 python 程式輸出 "票價為400元"，不是我要的，我希望:
1.18歲以下，不賣票
2.介於 18 ~ 60歲(不含)，賣400元
3.超過60歲，賣200元

(修改時，請微調我的程式就好)
------------------------
請給我修正後的程式:
age = 70
if (18 <= age):
    print('票價為400元')
elif (60 <= age):
    print('票價為200元')
else:
    print('不賣票')

[chatgpt]
你的程式有些小問題。讓我來微調幫你修正：

age = 70
if age < 18:
    print('不賣票')
elif age < 60:
    print('票價為400元')
else:
    print('票價為200元')


[我]
請指出你修改了哪裡

[chatgpt]

在你原本的程式中，條件判斷有些錯誤。這是你原本的程式碼：
age = 70
if (18 <= age):
    print('票價為400元')
elif (60 <= age):
    print('票價為200元')
else:
    print('不賣票')

我修改的部分在於條件判斷的方式。原本的程式中，第一個 if (18 <= age) 的判斷是對的，但第二個 elif (60 <= age) 的條件會在 18 歲以上的所有情況下都成立，因此不會達到預期的效果。修正後的程式修改了條件判斷的邏輯，確保了在不同年齡段下的正確票價。

[我]
請標出修改的地方

[chatgpt]
當然，這是修改後的程式碼：

age = 70
if age < 18:  # 修改此處的條件判斷方式
    print('不賣票')
elif age < 60:
    print('票價為400元')
else:
    print('票價為200元')

我修改了第一個條件判斷，將原本的 (18 <= age) 修改為 age < 18，以符合你所描述的需求。




多層的if判斷式:
用於判斷是否[達成幾次消費]的優惠條件。

例如:
>>>pointcard = True <--把True指派給pointcard變數，代表有集點卡。
>>>count = 5 <--把5指派給count 變數，代表消費5次。
>>>if (pointcard == True):<--確認pointcard變數，是否等於True。
>>>    if(count >= 5):<--確認count變數，是否大於、等於5。
>>>        print('兌換成功!票價為優惠200元')
        
>>>兌換成功!票價為優惠200元


自己舉例-如果未符合優惠條件，則使用別的價格:
>>>pointcard = True
>>>count = 4
>>>if (pointcard == True):
>>>    if(count >= 5):
>>>        print('兌換成功!票價為優惠200元')
>>>    else:
>>>        print('票價為400元')
        
>>>票價為400元



用and算符，合併條件
只要用一個if就能判斷[同時符合A條件和B條件]的情況。

例如:
>>>pointcard = True
>>>count = 5
>>>if (pointcard == True) and (count >= 5):
>>>    print('兌換成功!票價為優惠200元')
        
>>>兌換成功!票價為優惠200元




自己舉例-如果未符合優惠條件，則使用別的價格:
>>>pointcard = True
>>>count = 5
>>>if (pointcard == True) and (count >= 5):
    print('兌換成功!票價為優惠200元')
>>>else:
>>>    print('票價為400元')

>>>兌換成功!票價為優惠200元



賣電影票的集點卡優惠價語法，總整理:

例如:
>>>age = 35
>>>pointcard = True
>>>count = 4

>>>if (age < 18): <-先判斷是小於18歲，不賣票。
>>>    print('不賣票')
>>>elif (60 <= age): <-再判斷是否大於、等於60歲，有優惠價。
>>>    print('票價為200元')
>>>elif ((pointcard == True) and (count >= 5)): <-再同時判斷是否有集點卡和累積5點次數。
>>>    print('票價為200元')
>>>else: 
>>>    print('票價為400元')
    
>>>票價為400元





age = 35
pointcard = True
count = 4

if (age < 18): #先判斷是小於18歲，不賣票。
    print('不賣票')
elif ((60 <= age) or ((pointcard == True) and (count >= 5))): #同時判斷是大於60歲，或者有集點卡和累積5點次數。
    print('票價為200元')
else: 
    print('票價為400元')

