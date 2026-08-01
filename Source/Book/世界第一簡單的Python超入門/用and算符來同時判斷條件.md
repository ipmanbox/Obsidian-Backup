and算符，合併條件
只要用一個if就能判斷[同時符合A條件和B條件]的情況。

例如:
>>>pointcard = True <-有無集點卡
>>>count = 5 <-消費次數
>>>if (pointcard == True) and (count >= 5): ＜-用and串聯兩個條件，並同時判斷。
>>>    print('兌換成功!票價為優惠200元')      
>>>兌換成功!票價為優惠200元



自己舉例-如果未符合優惠條件，則使用別的價格:
>>>pointcard = True
>>>count = 5
>>>if (pointcard == True) and (count >= 5):
           print('兌換成功!票價為優惠200元')
>>>else:
>>>    print('票價為400元')




賣電影票的集點卡優惠價語法，總整理:

寫法1：
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

寫法2:
例如:
>>>age = 35
>>>pointcard = True
>>>count = 4

>>>if (age < 18): <-先判斷是小於18歲，不賣票。
>>>    print('不賣票')
>>>elif ((60 <= age) or ((pointcard == True) and (count >= 5))): <-同時判斷是大於60歲，或者有集點卡和累積5點次數。
>>>    print('票價為200元')
>>>else: 
>>>    print('票價為400元')


2024-01-09
19:14