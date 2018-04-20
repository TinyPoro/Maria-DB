[Nguồn](https://mariadb.com/kb/en/library/compound-composite-indexes/ "Permalink to Compound (Composite) Indexes - MariaDB Knowledge Base")

# Các chỉ mục hỗn hợp(composite) - kiến thức mariaDB cở sở,

## 1 bài học nhỏ trong "chỉ mục hỗn hợp" ("chỉ mục composite")

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
* chỉ mục(last_name, first_name) (1 chỉ mục "compound") 
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
* For the sake of simplicity, we can count each BTree lookup as 1 unit of work, and ignore scans for consecutive items. This approximates the number of disk hits for a large table in a busy system. 
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
5\. Phân phối kết quả (1865-1869). 
    
    
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

OK, so you get really smart and decide that MySQL should be smart enough to use both name indexes to get the answer. This is called "Intersect". 1\. Using INDEX(last_name), find 2 index entries with last_name = 'Johnson'; get (7, 17) 2\. Using INDEX(first_name), find 2 index entries with first_name = 'Andrew'; get (17, 36) 3\. "And" the two lists together (7,17) & (17,36) = (17) 4\. Reach into the data using seq = (17) to get the row for Andrew Johnson. 5\. Deliver the answer (1865-1869).
    
    
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
    

The EXPLAIN fails to give the gory details of how many rows collected from each index, etc.

## INDEX(last_name, first_name)

This is called a "compound" or "composite" index since it has more than one column. 1\. Drill down the BTree for the index to get to exactly the index row for Johnson+Andrew; get seq = (17). 2\. Reach into the data using seq = (17) to get the row for Andrew Johnson. 3\. Deliver the answer (1865-1869). This is much better. In fact this is usually the "best".
    
    
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
    

## "Covering": INDEX(last_name, first_name, term)

Surprise! We can actually do a little better. A "Covering" index is one in which _all_ of the fields of the SELECT are found in the index. It has the added bonus of not having to reach into the "data" to finish the task. 1\. Drill down the BTree for the index to get to exactly the index row for Johnson+Andrew; get seq = (17). 2\. Deliver the answer (1865-1869). The "data" BTree is not touched; this is an improvement over "compound".
    
    
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
    

Everything is similar to using "compound", except for the addition of "Using index".

## Variants

* What would happen if you shuffled the fields in the WHERE clause? Answer: The order of ANDed things does not matter. 
* What would happen if you shuffled the fields in the INDEX? Answer: It may make a huge difference. More in a minute. 
* What if there are extra fields on the the end? Answer: Minimal harm; possibly a lot of good (eg, 'covering'). 
* Reduncancy? That is, what if you have both of these: INDEX(a), INDEX(a,b)? Answer: Reduncy costs something on INSERTs; it is rarely useful for SELECTs. 
* Prefix? That is, INDEX(last_name(5). first_name(5)) Answer: Don't bother; it rarely helps, and often hurts. (The details are another topic.) 

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