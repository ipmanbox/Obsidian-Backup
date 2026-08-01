

1.[測試.pptx]的副檔名，改成[測試.zip]

2.打開[測試.zip]，在slides資料夾中(路徑:參考...\測試.zip\ppt\slides)
找到slide1.xml、slide2.xml、slide3.xml...以此類推

2.用記事本開啟[slide1.xml]檔案，用搜尋功能:
<p:transition 

3.在<p:transition 語法之內，添加以下語法:
advClick="0" advTm="3000">

整句可能會是這樣:
<p:transition spd="slow" advClick="0" advTm="3000"/>

備註:
3000=3千毫秒=3秒

4.儲存[slide1.xml]檔案，並放回[slides資料夾]中

5.全部改好之後，將[測試.zip]的副檔名，改回[測試.pptx]



#簡報修改 
#簡報技巧 
#ppt技巧 
#剪映課程 
