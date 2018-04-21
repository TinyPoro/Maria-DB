[Nguồn](https://mariadb.com/kb/en/library/compound-composite-indexes/ "Permalink to Compound (Composite) Indexes - MariaDB Knowledge Base")

# Các chỉ mục trôn(hỗn hợp) - kiến thức mariaDB cở sở,

## 1 bài học nhỏ trong "chỉ mục trộn" ("chỉ mục hỗn hợp")

Tài liệu này bắt đầu 1 cách tầm thường và có lẽ là chán ngắt, nhưng hướng tới nhiều thông tin thú vị hơn, có lẽ là những thứ bạn không nhận ra được về cách chỉ mục MariaDB và MySQL hoạt động thế nào.


Nó cũng giải thích [Giải thích][1] (cho vài phạm vi nhất định)

(Hầu hết những điều này cũng áp dụng cho các nhánh cơ sở dữ liệu không phải MySQl

## Câu truy vấn để thảo luận

Câu hỏi đặt ra là "Khi nào Andrew Johnson là tổng thống của Mĩ?".

Bảng `Presidents` có sẵn trông như thế này.
    
    
    +-----+------------+----------------+-----------+
    | seq | last_name  | first_name     | term      |
    +-----+------------+----------------+-----------+
    |   1 | Washington | George         | 1789-1797 |
    |   2 | Adams      | John           | 1797-1801 |
    ...
    |   7 | Jackson    | Andrew         | 1829-1837 |
    ...
    |  17 | Johnson    | Andrew         | 1865-1869 |
    ...
    |  36 | Johnson    | Lyndon B.      | 1963-1969 |
    ...
    

("Andrew Johnson" được lựa chọn cho bài học này là vì sự lặp lại của nó.)

(Các) chỉ mục nào sẽ là tốt nhất cho câu hỏi này? Cụ thể hơn, cái gì sẽ tốt nhất cho
    
    
        SELECT  term
            FROM  Presidents
            WHERE  last_name = 'Johnson'
              AND  first_name = 'Andrew';
    

1 vài chỉ mục INDEXes để thử...

* Không chỉ mục nào cả
* Chỉ mục(first_name), Chỉ mục(last_name) (2 chỉ mục riêng biệt) 
* "Index Merge Intersect" 
* chỉ mục(last_name, first_name) (1 chỉ mục "trộn") 
* chỉ mục(last_name, first_name, term) (1 chỉ mục "covering" ) 
* các biến thể 

## Không chỉ mục nào cả

Xem nào, tôi đang tào lao 1 chút ở đây. Tôi có 1 KHÓA CHÍNH  TRÊN `seq`, nhưng không nó không có 1 chút lợi thế nào trong câu truy vấn tôi đang học.
    
    
    mysql>  SHOW CREATE TABLE Presidents G
    CREATE TABLE `presidents` (
      `seq` tinyint(3) unsigned NOT NULL AUTO_INCREMENT,
      `last_name` varchar(30) NOT NULL,
      `first_name` varchar(30) NOT NULL,
      `term` varchar(9) NOT NULL,
      PRIMARY KEY (`seq`)
    ) ENGINE=InnoDB AUTO_INCREMENT=45 DEFAULT CHARSET=utf8
    
    mysql>  EXPLAIN  SELECT  term
                FROM  Presidents
                WHERE  last_name = 'Johnson'
                  AND  first_name = 'Andrew';
    +----+-------------+------------+------+---------------+------+---------+------+------+-------------+
    | id | select_type | table      | type | possible_keys | key  | key_len | ref  | rows | Extra       |
    +----+-------------+------------+------+---------------+------+---------+------+------+-------------+
    |  1 | SIMPLE      | Presidents | ALL  | NULL          | NULL | NULL    | NULL |   44 | Using where |
    +----+-------------+------------+------+---------------+------+---------+------+------+-------------+
    
    # Hoặc, sử dụng dạng hiển thị khác:  EXPLAIN ... G

               id: 1
      select_type: SIMPLE
            table: Presidents
             type: ALL        <-- Implies table scan
    possible_keys: NULL
              key: NULL       <-- Implies that no index is useful, hence table scan
          key_len: NULL
              ref: NULL
             rows: 44         <-- That's about how many rows in the table, so table scan
            Extra: Using where
    

## các chi tiết thực hiện

Đầu tiện, hãy miêu tả các InnoDB lưu trữ và sử dụng các chỉ mục.

* Dữ liệu và KHÓA CHÍNH được "nhóm lại" cùng nhau trong BTree.
* 1 tìm kiếm trên BTree thực sự nhanh và hiệu quả . Với 1 bảng 1 triệu dòng, có thể có 3 mực BTree, và 2 mực trên cùng có lẽ sẽ được cache
* Mỗi chỉ mục thứ cấp sẽ la 1 BTree khác, với khóa chính nhưu là 1 lá.
* Thu thập các phẩn tử ` liên tucj` ( dựa theo chỉ mục) từ BTree rất hiệu quả vì nó được lưu 1 cách liên tục.
* Để cho đơn giản hơn, chúng ta có thể coi mỗi tìm kiếm trên BTree như là 1 đơn vị công việc, và loại bỏ các lượt quét các thành phần liên tục. Điều này xấp xỉ số lần đĩa chọc vào bảng lớn trong hệ thống bận,

Vơi MyISAM, KHÓA CHÍNH không được lưu trong dữ liệu, vì vậy hay coi nó như là khóa thứ cấp (rất đơn giản)

## INDEX(first_name), INDEX(last_name)

Với người mới làm quen, một khi anh ta học được về đánh chỉ mục, sẽ quyết định đánh chỉ mục tất cả các cột, tất cả vào 1 lúc. Nhưng...

MySQL hiếm khi sử dùng nhiều hơn 1 chỉ mục 1 lúc trong 1 câu truy vấn. Vì vậy, 
nó sẽ phân tích các chỉ mục có thể
* first_name -- có 2 dòng khả dụng (1 tìm kiếm BTree , sau đó duyệt lần lượt) 
* last_name -- có 2 dòng khả dung. Hãy bảo nó chọn lastname. Đấy là các bước thực hiện việc SELECT: 
1\. Sử dụng INDEX(last_name), tìm 2 mục index với last_name = 'Johnson'. 
2\. Lấy PRIMARY KEY (ngầm thêm vào mỗi chỉ mục thứ cấp trong InnoDB); lấy (17, 36). 
3\. Tiếp cận dữ liệu sử dụng chuỗi = (17, 36)  để lấy các dòng cho Andrew Johnson và Lyndon B. Johnson.
4\.  Sử dụng phần còn lại của mệnh đề WHERE lọc tất cả trừ dòng mong muốn
5\. Xuất kết quả (1865-1869). 
    
    
    mysql>  EXPLAIN  SELECT  term
                FROM  Presidents
                WHERE  last_name = 'Johnson'
                  AND  first_name = 'Andrew'  G
      select_type: SIMPLE
            table: Presidents
             type: ref
    possible_keys: last_name, first_name
              key: last_name
          key_len: 92                 <-- VARCHAR(30) utf8 may need 2+3*30 bytes
              ref: const
             rows: 2                  <-- Two 'Johnson's
            Extra: Using where
    

## "Index Merge Intersect"

Được rồi, bạn sẽ trở trên thực sự thông minh và quyết định rằng MySQL nên đủ thông minh để sử dụng cả 2 tên chỉ mục để lấy kết quả. Điều này được gọi là "Interrsect.
1\. Sử dụng chỉ mục (last_name), tìm 2 phần tử chỉ mục với last_name = 'Johnson'; được (7,17)
2\. Sử dụng chỉ mục (first_name), tìm 2 phần tử chỉ mục với first_name = 'Andrew'; được(17,36)
3\. "Phép and" 2 danh sách với nhau (7,17) & (17,36) = (17)
4\. Tiếp cận dữ liệu sử dụng chuỗi = (17) để lấy dòng cho Andrew Johnson.
5\. Xuất kết quả (1865-1869). 
    
    
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: index_merge
    possible_keys: first_name,last_name
              key: first_name,last_name
          key_len: 92,92
              ref: NULL
             rows: 1
            Extra: Using intersect(first_name,last_name); Using where
    

Dòng giải thích quên không đưa tập thông tin về có bao nhiêu dòng được thu tập từ mỗi chỉ mục, vân vân.

## Chỉ mục(last_name, first_name)

Đây gọi là 1 chỉ mục "trộn" hoặc "hỗn hợp" vì nó có nhiều hơn 1 cột. 
1\. Tìm kiếm chỉ mục trong Btree để lấy chỉnh xác dòng chỉ mục cho Johnson+Andrew, được chuỗi = (17).
2\. Tiếp cận dữ liệu sử dụng chuỗi = (17) để lấy dòng Andrew Johnson.
3\. Xuất câu trả lời(1865-1869). Điều này tốt hơn rất nhiều. Sự thực là điều này thường là "tốt nhất".
    
    
        ALTER TABLE Presidents
            (drop old indexes and...)
            ADD INDEX compound(last_name, first_name);
    
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: ref
    possible_keys: compound
              key: compound
          key_len: 184             <-- The length of both fields
              ref: const,const     <-- The WHERE clause gave constants for both
             rows: 1               <-- Goodie!  It homed in on the one row.
            Extra: Using where
    

## "bao trùm": Chỉ mục(last_name, first_name, term)

Nhạc nhiên chưa! Chúng ta thực sự có thể làm tốt hơn 1 chút. 1 chỉ mục "Bao trùm" là 1 thứ mà _tất cả_ các trường của lệnh SELECT có thể được tìm thấy trong chỉ mục. Nó  có 1 điểm cộng thêm là không phải tiếp cận "dữ liệu"để hoàn thành công việc.
1\. Tìm kiếm chỉ mục trong BTre để lấy chỉ mục chính xác của dọng Johnson+Andrew, được chuỗi =(17).
2\. Xuất kết quả (1865-1869). "Dữ liệu" BTree không được động tới, đây là sự cỉa tiến so với "trộn".
    
    
        ... ADD INDEX covering(last_name, first_name, term);
    
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: ref
    possible_keys: covering
              key: covering
          key_len: 184
              ref: const,const
             rows: 1
            Extra: Using where; Using index   <-- Note
    

Mọi thứ tương tự như việc sử dụng "trộn", ngoài trừ việc thêm là "Sử dụng chỉ mục".

## Các biến thể

* Điều gì sẽ xảy ra nếu bạn xáo trộn các trường trong mệnh đề WHERE? Câu trả lời: Thứ tự mọi thứ trong AND không quan trọng.
* Điều gì sẽ xảy ra nếu bạn xáo trộn các trường trong Chỉ Mục ? Câu trả lời: Nó có thể tạo 1 sự khác biệt cực lớn. Có thể hơn 1 phút.
* Sẽ thế nào nếu có thêm trường vào cuối cùng? Câu trả lời: 1 chút ảnh hưởng nhỏ, có thể rất nhiều cái tốt ( ví dụ, "bao trùm").
* Thừa? Đó là, sẽ thế nào nếu bạn có cả 2 thứu này: Chỉ mục (a), Chỉ mục (a,b)? Cấu trà lời: Thừa chi phí gì đó trong các lệnh INSERT; nó hiếm khi có tác dụng trong các lệnh SELECT.
* Tiền tố? Đó là, Chỉ mục(last_name(5). first_name(5)) Câu trả lời: đừng bận tâm; nó hiếm khi có tác dụng, và thường làm hại. (chi tiết trong 1 chủ đề khác.)

## Thêm ví dụ:
    
    
        INDEX(last, first)
        ... WHERE last = '...' -- good (even though `first` is unused)
        ... WHERE first = '...' -- index is useless
    
        INDEX(first, last), INDEX(last, first)
        ... WHERE first = '...' -- 1st index is used
        ... WHERE last = '...' -- 2nd index is used
        ... WHERE first = '...' AND last = '...' -- either could be used equally well
    
        INDEX(last, first)
        Both of these are handled by that one INDEX:
        ... WHERE last = '...'
        ... WHERE last = '...' AND first = '...'
    
        INDEX(last), INDEX(last, first)
        In light of the above example, don't bother including INDEX(last).
    

## Postlog

Refreshed -- Oct, 2012; more links -- Nov 2016
ILLDI39.9KRankAgel0+1n/awhoissourceRankMore dataSummary reportDiagnosisDensity00n/a
Google +1
