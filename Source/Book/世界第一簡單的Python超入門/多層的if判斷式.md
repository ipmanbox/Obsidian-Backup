用於判斷是否[達成幾次消費]的優惠條件。

例如:
>>>pointcard = True <--把True指派給pointcard變數，代表有集點卡。
>>>count = 5 <--把5指派給count 變數，代表消費5次。
>>>if (pointcard == True):<--確認pointcard變數，是否等於True。
>>>    if(count >= 5):<--確認count變數，是否大於、等於5。
>>>        print('兌換成功!票價為優惠200元')
        
兌換成功!票價為優惠200元


自己舉例-如果未符合優惠條件，則使用別的價格:
>>>pointcard = True
>>>count = 4
>>>if (pointcard == True):
>>>    if(count >= 5):
>>>        print('兌換成功!票價為優惠200元')
>>>    else:
>>>        print('票價為400元')
        
票價為400元


2024-01-09
19:14